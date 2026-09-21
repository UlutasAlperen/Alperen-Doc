---
title: "docker-exec-shell"
weight: 8
---
# Exec

When it comes to _deploying_ applications with Docker, you'll usually just let the container do its thing. For example, the Ghost container we ran in the last chapter started up its own web server (based on the image configuration). We didn't need to run any manual commands in addition to just starting the container.

That said, it _is_ possible to run commands inside a running container! It's kinda like the container version of [ssh](https://www.ssh.com/ssh/)ing into a remote server and running a command.

List your running containers:

```bash
docker ps
```

Start up the "getting started" container again:

```bash
docker run -d -p 8965:80 docker/getting-started
```

Ensure that it's running:

```bash
docker ps
```

Run an `ls` command _from inside the container_ using the `docker exec` command:

```bash
docker exec CONTAINER_ID ls
```

Create a new `hacker.log` file in the working directory of the container by running `touch hacker.log` inside the container.
Run the `ls` command again to make sure that the file was created.

You should get a list of all the files and directories in the working directory (which happens to be the root in this case) of the container!

# Live Shell

Being able to run one-off commands is nice, but it's often more convenient to start a shell session running within the container itself. Thats where the `-i` and `-t` flags come in:

- `-i` makes the `exec` command interactive
- `-t` gives us a [tty (keyboard) interface](https://en.wikipedia.org/wiki/Tty_\(Unix\))
- Running `/bin/sh` gives us a shell session inside the container


```bash
docker exec -it CONTAINER_ID /bin/sh
```

Compose içinde komut çalıştırmak için [docker-compose-basic](../docker-compose-basic/) sayfasındaki `docker compose exec` bölümüne bak.
