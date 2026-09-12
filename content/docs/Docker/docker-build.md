---
title: "docker-build"
weight: 5
---
# Dockerfiles

Docker isn't _only_ useful for running _other_ people's software (as we've been doing so far). It's also a great way to build and package our own software.

![Container image composition](/images/docker/container-image-composition.png)

I've used Docker both ways. As a DevOps/platform engineer I'm usually using other's images, but as a backend developer I was usually building images for our own servers.

Docker images are built from _Dockerfiles_. A Dockerfile is just a text file that contains all the commands needed to assemble an image. It's essentially the ["Infrastructure as Code"](https://en.wikipedia.org/wiki/Infrastructure_as_Code) (IaC) for an image. It runs commands from top to bottom, kind of like a shell script.

Instead of manually installing dependencies on servers and making updates manually, we can check a Dockerfile into source control and build it automatically. Mhmmmm, _automation_

Create a file called `Dockerfile` in your working directory. If you're using VS Code, I'd recommend installing the [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker). It will give you some nice syntax highlighting.
Inside the Dockerfile add these lines of text:

```dockerfile
# This is a comment

# Use a lightweight debian os
# as the base image
FROM debian:stable-slim

# execute the 'echo "hello world"'
# command when the container runs
CMD ["echo", "hello world"]
```

Build a new image from the Dockerfile and call it `helloworld`:

```bash
docker build . -t helloworld:latest
```

The `-t helloworld:latest` flag tags the image with the name "helloworld" and the "latest" tag. Names are used to organize your images, and tags are used to keep track of different [versions](https://en.wikipedia.org/wiki/Software_versioning).

Run your image in a new container:

```bash
docker run helloworld
```

_If all went well, you'll see "hello world" printed to the console!_

Run `docker ps`. You'll notice that your container is _not_ running anymore! All it did was print and exit. Just like regular programs, docker containers can execute simple commands that exit quickly, or they can execute servers that run until killed. It just depends on the command you give it.
See the stopped container with `docker ps -a`.
Delete the Dockerfile, we don't need it anymore.


# Dockerizing the Server

Now that you know how to run your server manually, let's run it in Docker! The steps are simple:

1. Build the Server
2. Create a Dockerfile
3. Build an image using the Dockerfile (which will copy in the built server)
4. Run the image in a container

---

1. Create a `Dockerfile` in the root of your server's repo. Let's start with a simple lightweight [Debian Linux OS](https://www.debian.org/):

```dockerfile
FROM debian:stable-slim
```

2. Add a [`COPY`](https://docs.docker.com/engine/reference/builder/#copy) command on the next line of your `Dockerfile`. In the case of a simple compiled Go server, all we need is the compiled program itself!

```dockerfile
# COPY source destination
COPY goserver /bin/goserver
```

Replace "goserver" with the name of _your_ server executable if it's different.

The [ADD](https://docs.docker.com/engine/reference/builder/#add) command would also work here, but `COPY` is fine because we don't need the extra functionality that `ADD` offers.

3. Add a `CMD` command as the last line in the `Dockerfile`. This automatically starts the server process in the container when we run it.

```dockerfile
CMD ["/bin/goserver"]
```

4. Build your Dockerfile into an image.

```bash
docker build . -t goserver:latest
```

5. Start a new container from the image. Be sure to forward the ports to your host machine.

```bash
docker run -p 8010:8010 goserver
```

If you get an `exec format error`, it's probably because you built the go server for your local architecture, but you're trying to run it on a Linux OS! To fix it, rebuild the binary (and then the Dockerfile) with these flags:

> `GOOS=linux GOARCH=amd64 go build`

6. You should be able to access your server from the browser just like before, but this time it's running inside of Docker!
# Creating an Environment

You may be thinking, "What's the point of dockerizing this simple service"? Well, at the moment, there are only a couple of benefits:

- Anyone with Docker can run your image, regardless of their OS
- You can easily deploy containers of your image on any cloud service that uses images (most of them) or on an orchestration server like [Kubernetes](https://kubernetes.io/).
- If your server were written in a language like Python or JavaScript, you could bundle the interpreter and dependencies inside the image so that you don't need to reconfigure them on the server.

That said, because our app is so simple, there's just not much environment required, and one of the best things about Docker is that it allows you to ship an entire environment.

So... let's make it more interesting!

We're going to make the port that our server binds to configurable: it will be set by an environment variable.

1. Find the line that sets `port` to a hard-coded value of `8010` and update it so that it reads an environment variable called `PORT`. You can use [`os.Getenv`](https://pkg.go.dev/os#Getenv):

```go
port := os.Getenv("PORT")
```

Make sure that the `os` package is imported:

```go
import (
	"fmt"
	"log"
	"net/http"
	"os"
	"time"
)
```

2. Change the port to 8999 by setting an environment variable in your shell:

```bash
export PORT="8999"
```

3. Rebuild and run your Go program and make sure it serves on port 8999. Remember to replace `goserver` with the name of your binary.

```bash
go build
./goserver
```

4. Add an [ENV command](https://docs.docker.com/engine/reference/builder/#env) to your Dockerfile to set the port within the image. You'll need to do it _before_ the `CMD` command so that the environment variable is set before the server starts.

```dockerfile
ENV PORT=8991
```

5. Rebuild your Docker image:

```bash
docker build . -t goserver:latest
```

6. Rerun your Docker container, be sure to expose the correct port:

```bash
docker run -p 8991:8991 goserver
```

### example for js file 

Start by reading the application code to understand its configuration and signal handling.

The `index.js` file reveals two important details:

1. The app requires `PORT`, `LOG_LEVEL`, and `ENVIRONMENT` environment variables and exits with an error if any of them is missing.
2. It registers a graceful shutdown handler specifically for `SIGINT` (not the Docker's default `SIGTERM`).


```dockerfile
FROM node:24-slim
WORKDIR /app

COPY index.js .

ENV PORT=8080
ENV LOG_LEVEL=debug
ENV ENVIRONMENT=development

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
USER appuser

STOPSIGNAL SIGINT

CMD ["node", "index.js"]
```
Here's what each section does:

- `ENV`'s set default values for the configuration variables. These become baked into the image and are used unless overridden at runtime with `docker run -e`.
- `USER` switches to a non-root user. The `adduser --system` creates a system user without a home directory or login shell - appropriate for service accounts.
- `STOPSIGNAL` tells Docker to send `SIGINT` (instead of the default `SIGTERM`) when `docker stop` is called. This matches the signal the application actually handles for graceful shutdown.
