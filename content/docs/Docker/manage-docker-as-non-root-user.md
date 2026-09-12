---
title: "manage-docker-as-non-root-user"
weight: 2
---
## [Manage Docker as a non-root user](https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)

The Docker daemon binds to a Unix socket, not a TCP port. By default it's the `root` user that owns the Unix socket, and other users can only access it using `sudo`. The Docker daemon always runs as the `root` user.

If you don't want to preface the `docker` command with `sudo`, create a Unix group called `docker` and add users to it. When the Docker daemon starts, it creates a Unix socket accessible by members of the `docker` group. On some Linux distributions, the system automatically creates this group when installing Docker Engine using a package manager. In that case, there is no need for you to manually create the group.

> Warning
> 
> The `docker` group grants root-level privileges to the user. For details on how this impacts security in your system, see [Docker Daemon Attack Surface](https://docs.docker.com/engine/security/#docker-daemon-attack-surface).

> Note
> 
> To run Docker without root privileges, see [Run the Docker daemon as a non-root user (Rootless mode)](https://docs.docker.com/engine/security/rootless/).

To create the `docker` group and add your user:

1. Create the `docker` group.

```bash
sudo groupadd docker
```
2. Add your user to the `docker` group.

```bash
sudo usermod -aG docker $USER
```
3. Log out and log back in so that your group membership is re-evaluated.

> If you're running Linux in a virtual machine, it may be necessary to restart the virtual machine for changes to take effect.

You can also run the following command to activate the changes to groups:
    
```bash
 newgrp docker
```

3. Verify that you can run `docker` commands without `sudo`.

```bash
 docker run hello-world
```

  This command downloads a test image and runs it in a container. When the container runs, it prints a message and exits.
  
  If you initially ran Docker CLI commands using `sudo` before adding your user to the `docker` group, you may see the following error:
  
```md
WARNING: Error loading config file: /home/user/.docker/config.json -
stat /home/user/.docker/config.json: permission denied
```

This error indicates that the permission settings for the `~/.docker/` directory are incorrect, due to having used the `sudo` command earlier.
	
  To fix this problem, either remove the `~/.docker/` directory (it's recreated automatically, but any custom settings are lost), or change its ownership and permissions using the following commands:
  
```bash
 sudo chown "$USER":"$USER" /home/"$USER"/.docker -R
 sudo chmod g+rwx "$HOME/.docker" -R
```

## [Configure Docker to start on boot with systemd](https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot-with-systemd)

Many modern Linux distributions use [systemd](https://systemd.io/) to manage which services start when the system boots. On Debian and Ubuntu, the Docker service starts on boot by default. To automatically start Docker and containerd on boot for other Linux distributions using systemd, run the following commands:

```bash
 sudo systemctl enable docker.service
 sudo systemctl enable containerd.service
```

To stop this behavior, use `disable` instead.

```bash
 sudo systemctl disable docker.service
 sudo systemctl disable containerd.service
```