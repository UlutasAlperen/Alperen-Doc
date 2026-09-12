---
title: "docker-logs"
weight: 9
---
# Docker Logs

When containers are running in detached mode with the `-d` flag, you don't see any output in your terminal, which is nice for keeping your terminal clean, but what if something goes _wrong_?

_Enter the `docker logs` command_.

```bash
docker logs [OPTIONS] CONTAINER
```

## Ornek log

1. Let's run the Linux `alpine` image in a new container in detached mode, and give it a simple command to run to generate some standard output:

```bash
docker run -d --name logdate alpine sh -c 'while true; do echo "LOGGING: $(date)"; sleep 1; done'
```

The `sh -c 'while true; do echo "LOGGING: $(date)"; sleep 1; done'` part is just a simple shell script to execute inside the container that prints the current date and time every second.

2. Find the ID of the running container with `docker ps`.
3. Use the `docker logs` command to view the logs of the container.

Notice that if you run `docker logs` over and over again, you will get different output. That's because you're only getting the most recent logs each time.

4. Add the `-f` flag to the `docker logs` command to follow the logs in real-time.
5. Exit the logs with `ctrl+c`, then view only the most recent 5 logs with the `--tail` option:

```bash
docker logs --tail 5 CONTAINER
```

# Stats

Okay, we know how to inspect a containers logs, but what if we want to see the resource utilization?

It's common to spin up some Docker containers, forget about them, and then wonder why your host machine has gotten really slow. It's really nice to see how much RAM/CPU each container is using, and it's _critical_ in production environments.

The `docker stats` command gives you a live data stream of resource usage for running containers.

```bash
docker stats [OPTIONS] [CONTAINER...]
```

## PRACTICE

1. The pre-built [stress-ng](https://hub.docker.com/r/alexeiled/stress-ng) image is a nice little image that we can use to create a container that artificially uses a lot of CPU/memory resources. Start a container that uses a full CPU core:

```bash
docker run -d --name cpu-stress alexeiled/stress-ng --cpu 2 --timeout 10m
```

This is going to slow down your machine somewhat while it's running, but don't worry we'll kill it soon. We added a timeout of 10 minutes just in case you forget to kill it later.

2. Start another container that uses some memory:

```bash
docker run -d --name mem-stress alexeiled/stress-ng --vm 1 --vm-bytes 1G --timeout 10m
```

This one will allocate a full gigabyte of memory for the container, which is a lot for a single container. Again, we'll kill it soon.

3. Run `docker stats` to see the live resource usage of the two containers. _Take a good look_! You should see a table with CPU, memory, network I/O, and block I/O usage for each container. Press Ctrl+C to exit the stats view.

```bash
docker stats
```
# Top

The `docker top` command shows the running _processes inside_ a container.

```bash
docker top CONTAINER [ps OPTIONS]
```
# Resource Limits

When you notice a container's using too many resources, if you don't have the time or the ability to "fix" the code, you can limit the resources the container has available. The `docker run` command has a few options for limiting resources:

- `--memory`: Limit the memory available to the container
- `--cpus`: Limit the CPU shares available to the container

```bash
 docker run -d --cpus="0.25" --name cpu-stress alexeiled/stress-ng --cpu 2 --timeout 10m
```