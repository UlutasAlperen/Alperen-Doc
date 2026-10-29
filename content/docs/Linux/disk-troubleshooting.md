---
title: "disk-troubleshooting"
weight: 160
---
**Scope:** Diagnosing and resolving full-filesystem and inode exhaustion on Debian-based and RHEL-based distributions  
**Topics:** df, du, inodes, deleted-but-open files, lsof, log growth, package caches, containers.

## 1. Check Space Usage with df

Start with the filesystem that reports high usage:

```bash
df -h
df -hT
```

Expected output (example):

```text
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sda2      ext4   50G   48G  2.0G  96% /
```

**Options:**
- `-h`: Human-readable sizes.
- `-T`: Adds the filesystem type.

A full filesystem fails writes with `No space left on device`. Note the mount point: a full `/var` can stop services while `/` still has free space.

## 2. Check Inode Usage with df -i

Check the inode table:

```bash
df -i
```

Expected output (example):

```text
Filesystem      Inodes  IUsed   IFree IUse% Mounted on
/dev/sda2      3276800 3276800      0  100% /
```

Every file and directory consumes an inode. ext4 reserves a fixed inode count at `mkfs` time, so a filesystem can be exhausted while `df -h` still shows free space. The error message is the same: `No space left on device`.

Resolve inode exhaustion by deleting many small files; inodes are freed with their files. ext4 cannot gain inodes without re-creating the filesystem. XFS allocates inodes dynamically and is not affected by this limit.

## 3. Find Large Directories with du

Summarize the largest directories on one filesystem:

```bash
du -xh -d 1 / | sort -h | tail -n 20
```

Drill down into the result:

```bash
du -xh -d 1 /var | sort -h | tail -n 10
du -sh /var/log/*
```

**Options:**
- `-x`: Stays on a single filesystem and does not descend into other mounts.
- `-d 1`: Limits the depth to one level.
- `sort -h | tail`: Puts the largest directories last.

Compare `du` against `df`: `du` summarizes file sizes by path, `df` reports filesystem usage. Large differences point to deleted-but-open files.

## 4. Find Deleted-but-Open Files with lsof

A file deleted while a process still holds it open keeps its space until the process releases the descriptor. This is why `df` reports a full filesystem while `du` finds nothing large.

List such files:

```bash
lsof +L1
```

Expected output (example):

```text
COMMAND  PID USER FD   TYPE DEVICE SIZE/OFF NLINK NODE
nginx    1234 root 2w  REG  253,2  4500000000  0  12345 /var/log/nginx/access.log
```

Free the space by restarting the service that holds the file:

```bash
systemctl restart <SERVICE>
```

Note: `rm` on an open file does not free space immediately. Truncating does: `: > <FILE>`. Check open handles before deleting anything.

## 5. Inspect Common Culprits

| Location | Typical cause | First check |
|---|---|---|
| `/var/log` | Growing syslog, audit, or application logs | `du -xh -d 1 /var/log` |
| Journal storage | Large systemd journal | `journalctl --disk-usage` |
| Package caches | Downloaded packages | `du -sh /var/cache/apt` or `/var/cache/dnf` |
| Containers | Image layers and container logs | `docker system df` |
| `/tmp`, `/var/tmp` | Leftover build artifacts | `du -xh -d 1 /tmp` |
| `/home` | User data, archives, databases | `du -xh -d 1 /home` |

Container storage adds another layer: image layers live under `/var/lib/docker` (or `/var/lib/containers`) on an overlay filesystem. `df` reports the underlying mount, so check `docker system df` and the size of `/var/lib/docker/containers/*/*-json.log` separately.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Clean the APT Cache

Inspect the cache size:

```bash
du -sh /var/cache/apt/archives
```

Remove downloaded packages:

```bash
apt clean
```

Remove packages that can no longer be downloaded:

```bash
apt autoclean
```

Remove automatically installed dependencies that are no longer required:

```bash
apt autoremove
```

## 2. Reduce Journal Growth

Check current usage:

```bash
journalctl --disk-usage
```

Vacuum the journal to a maximum size:

```bash
journalctl --vacuum-size=200M
```

Or by age:

```bash
journalctl --vacuum-time=7d
```

**Options:**
- `--vacuum-size`: Keeps the journal below the given size.
- `--vacuum-time`: Removes entries older than the given time span.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Clean the DNF Cache

Inspect and clean:

```bash
du -sh /var/cache/dnf
dnf clean all
```

Remove unneeded packages:

```bash
dnf autoremove
```

## 2. Configure Persistent Journal Storage

By default the journal lives in `/run/log/journal` and is lost at reboot. Enable persistent storage and cap its size in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=500M
```

Apply the configuration:

```bash
mkdir -p /var/log/journal
systemctl restart systemd-journald
journalctl --disk-usage
```

**Options:**
- `Storage=persistent`: Stores the journal in `/var/log/journal` across reboots.
- `SystemMaxUse`: Caps total journal disk usage.

---

# Part 3: Important Notes

## Symptom to Command

| Symptom | Command |
|---|---|
| Writes fail with `No space left on device` | `df -h` |
| `df -h` shows free space but writes fail | `df -i` |
| `df` and `du` disagree | `lsof +L1` |
| Disk fills continuously | `journalctl --disk-usage`, `du -xh -d 1 /var` |
| Full after package updates | `apt clean` / `dnf clean all` |
| Full after container activity | `docker system df` |
| Unknown large directory | `du -xh -d 1 <MOUNT_POINT> \| sort -h \| tail` |
| Full filesystem with few large files | Delete many small files, then recheck `df -i` |

## Security Guidelines

- Never delete files in `/var/log` blindly. Truncate them (`: > <FILE>`) or let logrotate handle them; a deleted-but-open log keeps its space.
- Run `lsof +L1` before deleting files during an incident to understand who holds them open.
- Do not delete package-manager state in `/var/lib/dpkg` or `/var/lib/rpm`, or container data in `/var/lib/docker`, by hand. Use the owning tool (`apt`, `dnf`, `docker system prune`).
- Keep at least 10-15% free space on each filesystem so logs and temporary files can still be written during incidents.
- Restrict access to logs and diagnostic output; they can contain credentials and personal data.
- Run destructive cleanup commands as root only after confirming the target path with `ls` or `du`.
- Prefer capping growth (`SystemMaxUse`, logrotate, application log settings) over periodic manual deletion.
- Investigate sudden disk growth as a potential compromise: dropped payloads, log flooding, or hidden data are common signs.
