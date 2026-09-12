---
title: "docker-containers-images"
weight: 3
---
# Containers

> A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.
>
> -- [Docker](https://www.docker.com/resources/what-container/)

Containers, on the other hand, gives us 90% of the benefits of virtual machines, but are _super_ lightweight. _Containers boot up in seconds, while virtual machines can take minutes._

## Virtual Machine Architecture

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/QS4HGNG.png)

## Container (Docker) Architectures

![](https://storage.googleapis.com/qvault-webapp-dynamic-assets/course_assets/fmZG1Zd-662x400.png)

## Why Are Containers Lightweight?

Virtual machines virtualize _hardware_, they emulate what a physical computer does at a low level. Containers virtualize at the _operating system_ level. Isolation between containers that are running on the same machine is still _really good_. For the most part, each container _feels like_ it has its own operating and filesystem. In reality, a lot of resources are being shared, but they're being shared securely through [namespaces](https://docs.docker.com/engine/security/userns-remap/).

Your browser does not support playing HTML5 video. You can instead. Here is a description of the content: containers vs virtual machines

# Images

So a "container" is kinda like a lightweight VM, great... so what's an _image_?

- **Image**: A read-only _definition_ of a container
- **Container**: an _instance_ of a virtualized read-write environment

A container is basically an image that's **actively running**. In other words, you boot up a container _from_ an image. You can create multiple separate containers all from the same image (it's _kinda_ like the relationship between classes and objects).

>pull to your local machine
```bash
docker pull docker/getting-started
```

>see what u pull your local machine
```bash
docker images
```
# Run a Container

Now that you've downloaded the [getting started image](https://hub.docker.com/r/docker/getting-started), let's _run_ it inside a new container. The `docker run` command starts a new container from an image. Let's break down the syntax:

```bash
# this is just an example, don't run this
docker run -d -p hostport:containerport namespace/name:tag
```

- `-d`: Run in detached mode (doesn't block your terminal)
- `-p`: Publish a container's port to the host (forwarding)
- `hostport`: The port on your local machine
- `containerport`: The port inside the container
- `namespace/name`: The name of the image (usually in the format `username/repo`)
- `tag`: The version of the image (often `latest`)

1. Use the `run` command to start a new container from the "getting started" image:

```bash
docker run -d -p 8965:80 docker/getting-started:latest
```

2. You should see the container running in the "Containers" tab of Docker Desktop. Run this on the command line to see the running containers:

```bash
docker ps
```

On one of the columns you should see this:

```
PORTS
0.0.0.0:8965->80/tcp
```

This is saying that port `8965` on your local "host" machine is being forwarded to port `80` on the running container. Port `80` is conventionally used for HTTP web traffic. Navigate to `http://localhost:8965` and you should see a webpage served from the container!

**Run and submit** the CLI tests **against your "getting started" container on `http://localhost:8965`**.

# Stop a Container

Okay, now you know how to start a new instance of a container from an image... but how do you _stop_ it? There are two primary ways:

- `docker stop`: This stops the container by issuing a `SIGTERM` signal to the container. You'll typically want to use `docker stop`.
- `docker kill`: This stops the container by issuing a `SIGKILL` signal to the container. This is a more forceful way to stop a container, and should be used as a last resort.

```bash
for id in $(docker ps -q); do
    docker stop "$id"
done
```

For persistent data, see [docker-volumes](docker-volumes/).
