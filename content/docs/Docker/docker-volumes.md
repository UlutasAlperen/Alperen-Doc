---
title: "docker-volumes"
weight: 4
---
# Volumes

By default, Docker containers don't retain any state from _past_ containers. For example, if I:

1. Start a container from an image
2. Make some changes to the filesystem (like installing a new package) in that container
3. Stop the container
4. Start a new container from the same image
5. The new container does _not_ have the changes I made in step 2.

However, if I restart the _stopped_ container, it _will_ have the changes I made. This is only worth mentioning because sometimes developers think that killing an old container and starting a new one is the same as _restarting a process_ - but that's not true... it's more like resetting the state of the _entire machine_ to the original image.

All this said, Docker _does_ have ways to support "persistent state" through [storage volumes](https://docs.docker.com/storage/volumes/). They're basically a filesystem that lives outside of the container, but can be accessed by the container.


 Create a new empty volume called `ghost-vol`:

```bash
docker volume create ghost-vol
```

 Make sure it worked:

```bash
docker volume ls
```

 Inspect the volume to see where it is on your local machine:

```bash
docker volume inspect ghost-vol
```

Docker hosts an official image for Ghost on [Docker Hub](https://hub.docker.com/_/ghost).

 Pull the Ghost image from Docker Hub:

```bash
docker pull ghost
```

Run the Ghost image in a new container:

```bash
docker run -d -e NODE_ENV=development -e url=http://localhost:3001 -p 3001:2368 -v ghost-vol:/var/lib/ghost ghost
```

- `-d` runs the image in detached mode to avoid blocking the terminal.
- `-e NODE_ENV=development` sets an [environment variable](https://en.wikipedia.org/wiki/Environment_variable) within the container. This tells Ghost to run in "development" mode (rather than "production", for instance)
- `-e url=http://localhost:3001` sets another environment variable, this one tells Ghost that we want to be able to access Ghost via a URL on our host machine.
- We've used `-p` before. `-p 3001:2368` does some [port-forwarding](https://en.wikipedia.org/wiki/Port_forwarding) between the container and our host machine.
- `-v ghost-vol:/var/lib/ghost` mounts the `ghost-vol` volume that we created before to the `/var/lib/ghost` path in the container. Ghost will use the `/var/lib/ghost` directory to persist stateful data (files) between runs.

3. Navigate to `http://localhost:3001/` in your browser, you should see your new Ghost CMS!

## Remove docker volume

Use `docker ps -a` to see _all_ containers, even those that aren't running.
Stop the running Ghost container
Remove the ghost container. Use `docker --help` to find the right command.
Remove the `ghost-vol` volume. Use `docker volume --help` to find the right command.
Now that it's gone, let's see what happens if we try to start the Ghost container back up and attach it to a volume that doesn't exist.

```bash
docker run -d -e NODE_ENV=development -e url=http://localhost:3001 -p 3001:2368 -v ghost-vol:/var/lib/ghost ghost
```

Navigate to `http://localhost:3001/` in your browser, and you should see a fresh CMS. That's weird, why no errors?

Run:

```bash
docker volume ls
```

The `ghost-vol` is back from the dead!?! It turns out the `-v ghost-vol:/var/lib/ghost` flag binds to a "ghost-vol" volume if it exists, otherwise, it creates it automatically!

So, we now have a fresh installation. Our post that was on the old volume is gone, but this new volume will persist if we don't delete it.


Bu çıktı, `docker ps -a` listesidir.
İçinde:

- **İlk satır (62a79dd0b7e4)** → aktif çalışan container (`Up 11 minutes`).
- Diğerleri → durdurulmuş (Exited).

Tümünü - çalışan ve durdurulmuş dahil - sadece ID olarak bastırmak için:

```bash
docker ps -aq
```

Silmek istersek:

- Sadece durdurlumus state'de olanlar:
```bash
docker ps -aq -f status=exited | xargs docker rm
```

- Hepsi :
```bash
docker rm -f $(docker ps -aq)
```

Bu ikinci komut tüm container’ları  durdurur ve siler.

Var olan bir volume üzerine compose ile devam etmek için [var-olan-volume-uzerine-docker-compose](var-olan-volume-uzerine-docker-compose/) sayfasına bak.
