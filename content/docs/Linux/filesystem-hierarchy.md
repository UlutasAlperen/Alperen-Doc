---
title: "filesystem-hierarchy"
weight: 130
---
**Scope:** Standard Linux filesystem hierarchy on Debian-based and RHEL-based distributions  
**Topics:** FHS directory layout, /bin vs /sbin, virtual filesystems, mount points, safe cleanup, and reference tables.

## 1. Understand the Root Directory

The root directory `/` is the single top of the filesystem tree. Every file and device on the system is reachable from it.

Inspect it:

```bash
ls /
```

Expected output (contents vary by installation):

```text
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
```

The layout follows the Filesystem Hierarchy Standard (FHS). Each directory holds a specific category of data: configuration in `/etc`, variable data in `/var`, user data in `/home`, and virtual kernel interfaces in `/proc`, `/sys`, and `/dev`.

## 2. Distinguish /bin and /sbin

- `/bin`: Essential user commands required for system operation and recovery (`ls`, `cp`, `cat`, `mount`).
- `/sbin`: System administration commands intended for root (`fdisk`, `mkfs`, `reboot`).

On Debian 12 and RHEL 9 both directories are merged into `/usr`:

```bash
ls -ld /bin /sbin
```

Expected output:

```text
lrwxrwxrwx 1 root root 7 ... /bin -> usr/bin
lrwxrwxrwx 1 root root 7 ... /sbin -> usr/sbin
```

The distinction is preserved through the symbolic links. Unprivileged users often do not have `/sbin` in `PATH`, and most commands there require root privileges anyway.

## 3. Review /etc

`/etc` contains host-specific configuration as plain text files. No binaries belong here. Examples:

- `/etc/fstab`: Filesystem mount table.
- `/etc/ssh/sshd_config`: OpenSSH server configuration.
- `/etc/passwd`, `/etc/group`, `/etc/shadow`: User and group databases.
- `/etc/systemd/system`: Local systemd unit overrides.

Back up `/etc` separately. Configuration is not recoverable from package reinstallation alone.

## 4. Review /var

`/var` holds data that changes while the system is running.

| Path | Contents |
|---|---|
| `/var/log` | Log files and journal storage |
| `/var/lib` | Persistent application state (package databases, containers, services) |
| `/var/cache` | Rebuildable cached data (package downloads) |
| `/var/spool` | Queued work (print jobs, mail, cron) |
| `/var/tmp` | Temporary files preserved across reboots |

`/var` is the usual candidate for a separate partition or logical volume because its growth is independent of `/usr` and `/etc`.

## 5. Review /home and /root

- `/home/<USERNAME>`: Per-user data, shell profiles, and SSH keys.
- `/root`: Home directory of the root account. It is not located under `/home` so that `/home` can be mounted from separate storage.

Keep user data out of system directories. Install quotas on `/home` when multiple users share the host.

## 6. Review /tmp and /run

- `/tmp`: World-writable scratch space with the sticky bit (`1777`). Contents are removed at boot. On RHEL-based systems it is typically a `tmpfs` mount; on Debian-based systems it usually resides on the root filesystem.
- `/run`: `tmpfs` for volatile runtime state (PID files, sockets, lock files). Cleared at every boot.

Inspect the mounts:

```bash
findmnt /run
findmnt /tmp
```

Expected output for `/run`:

```text
TARGET SOURCE FSTYPE OPTIONS
/run   run    tmpfs  rw,nosuid,nodev,noexec,relatime,size=...
```

The sticky bit on `/tmp` allows everyone to create files but only the owner (or root) can delete them. Do not store durable data in either directory. Systemd services with `PrivateTmp=yes` receive a private `/tmp` that is invisible to other processes.

## 7. Review /usr

`/usr` contains read-mostly, shareable data installed by the package manager:

- `/usr/bin`, `/usr/sbin`: Binaries.
- `/usr/lib`, `/usr/lib64`: Libraries.
- `/usr/share`: Architecture-independent data (documentation, icons, timezone data).
- `/usr/local`: Locally compiled or manually installed software (`/usr/local/bin`, `/usr/local/lib`). Package managers never touch it.

Do not edit package-managed files under `/usr` directly; upgrades overwrite them. Use `/usr/local` for manual installs.

## 8. Review /opt and /srv

- `/opt`: Self-contained third-party application bundles, for example `/opt/<APP>/bin/<APP>`.
- `/srv`: Data served by the system, for example `/srv/www` or `/srv/ftp`.

Give each service its own subdirectory and own it with the service account.

## 9. Review /mnt and /media

- `/mnt`: Temporary manual mounts created by the administrator.
- `/media`: Automounted removable media (USB drives, optical media) on desktop systems.

Do not store persistent data in these directories. For permanent storage, create a dedicated mount point and register it in `/etc/fstab`.

## 10. Inspect the Virtual Filesystems

`/proc`, `/sys`, and `/dev` are kernel interfaces generated at runtime. They consume no real disk space.

```bash
findmnt -t proc,sysfs,devtmpfs
```

- `/proc` (procfs): Process and kernel information (`/proc/cpuinfo`, `/proc/meminfo`, `/proc/<PID>/status`).
- `/sys` (sysfs): Device and driver model (`/sys/block`, `/sys/class/net`).
- `/dev` (devtmpfs): Device nodes (`/dev/sda`, `/dev/nvme0n1`, `/dev/null`).

Files under these trees are generated on read. Never delete them, copy them recursively, or back them up as ordinary data.

## 11. Review /boot

`/boot` holds the kernel, initial RAM disk, and boot loader configuration. Typical contents:

- `vmlinuz-<VERSION>`: Compressed kernel image.
- `initrd.img-<VERSION>` (Debian) or `initramfs-<VERSION>.img` (RHEL): Initial RAM disk.
- `grub/grub.cfg` (Debian) or `grub2/grub.cfg` (RHEL): Boot loader configuration.

Keep the running kernel and at least one known-good kernel. Deleting files here can make the system unbootable. Monitor free space when `/boot` is a separate small partition.

## 12. Inspect Mounts with findmnt

`findmnt` displays the mount tree as the kernel sees it:

```bash
findmnt
findmnt --target /var
findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS
```

Expected output for `findmnt --target /var`:

```text
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/sda2 ext4   rw,relatime
```

Conceptually, a mount overlays a filesystem onto a directory of the parent filesystem. The kernel resolves paths across mount boundaries, which is why a full `/var` partition can stop writes while `/` still reports free space.

---

# Part 3: Important Notes

## Directory Reference

| Path | Purpose | Example contents |
|---|---|---|
| `/` | Root of the filesystem tree | `bin`, `etc`, `usr`, `var` |
| `/bin` | Essential user commands | `ls`, `cp`, `cat` |
| `/sbin` | System administration commands | `fdisk`, `mkfs`, `reboot` |
| `/etc` | Host configuration files | `fstab`, `passwd`, `ssh/sshd_config` |
| `/var` | Variable runtime data | `log`, `lib`, `cache`, `spool` |
| `/home` | User home directories | `/home/<USERNAME>` |
| `/root` | Root account home directory | `.bashrc`, `.ssh` |
| `/tmp` | Temporary scratch space | Build artifacts, session files |
| `/run` | Volatile runtime state | PID files, sockets |
| `/usr` | Read-mostly installed software | `bin`, `lib`, `share` |
| `/opt` | Third-party application bundles | `/opt/<APP>/bin` |
| `/srv` | Served data | `www`, `ftp` |
| `/mnt` | Temporary manual mounts | Ad hoc device mounts |
| `/media` | Removable media automounts | USB drives |
| `/proc` | Process and kernel information | `cpuinfo`, `<PID>/status` |
| `/sys` | Device and driver model | `block`, `class/net` |
| `/dev` | Device nodes | `sda`, `nvme0n1`, `null` |
| `/boot` | Kernel and boot loader | `vmlinuz-<VERSION>`, `grub.cfg` |

## Cleanup Safety

| Action | Verdict | Reason |
|---|---|---|
| Clear package files in `/var/cache` | Safe | Data is rebuildable |
| Truncate old logs in `/var/log` | Safe with rotation | Recent logs are needed for diagnostics |
| Delete user files in `/home` | Only with the owner | User data is not reproducible |
| Delete files in `/tmp` and `/var/tmp` | Safe after checking running processes | Scratch data, cleared over time |
| Delete anything in `/proc`, `/sys`, `/dev` | Never | Virtual kernel interfaces |
| Delete configuration in `/etc` | Never | Required to operate the system |
| Delete contents of `/boot` | Never | Makes the system unbootable |
| Edit package-managed files in `/usr` | Never | Overwritten by package upgrades |

## Immutable Parts of the Hierarchy

- `/proc`, `/sys`, and `/dev` are virtual. Their contents reflect live kernel state and cannot be removed.
- `/usr` is owned by the package manager. Locally installed software belongs in `/usr/local`.
- `/boot` is managed by kernel packages and boot loader tooling. Use those tools instead of manual deletion.
- `/etc/fstab` and the running mount tree must stay consistent. A wrong entry can leave the system in emergency mode at boot.

## Security Guidelines

- Keep `/tmp` and `/var/tmp` mounted with the sticky bit (`1777`) and the `nodev`, `nosuid` options.
- Restrict `/root` to mode `0700` and never share it between users.
- Never make system directories world-writable. `chmod 777` on `/etc`, `/usr`, or `/var` breaks the security model.
- Protect `/etc` backups like the live system; they contain password hashes and private keys.
- Mount `/home`, `/var`, or `/opt` from separate storage with appropriate options (`nodev` on `/home`).
- Install local binaries in `/usr/local` and review their permissions before adding the directory to `PATH`.
- Monitor mount points with `findmnt` and `df -h` to detect unexpected mounts or unexpectedly full filesystems.
- Treat unexpected new top-level directories as suspicious during incident review.
