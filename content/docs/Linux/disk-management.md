---
title: "disk-management"
weight: 140
---
**Scope:** Block devices, partitions, filesystems, mounts, fstab, and swap on Debian-based and RHEL-based distributions  
**Topics:** lsblk, blkid, parted, GPT, mkfs.ext4, mkfs.xfs, mount, fstab, swap.

## 1. Inspect Block Devices

Identify every block device before touching it:

```bash
lsblk
lsblk -f
```

Expected output (example):

```text
NAME   FSTYPE LABEL UUID                                 MOUNTPOINT
sda
├─sda1 ext4         1a2b3c4d-0000-0000-0000-000000000000 /
└─sda2 swap         5e6f7a8b-0000-0000-0000-000000000000 [SWAP]
sdb
└─sdb1 ext4         9c8d7e6f-0000-0000-0000-000000000000 /srv/data
```

**Options:**
- `lsblk`: Lists devices, sizes, types, and mount points.
- `-f`: Adds filesystem type, label, and UUID.

## 2. Read Device Attributes with blkid

Inspect the usage:

```bash
blkid /dev/<DEVICE>
blkid /dev/<DEVICE>1
```

Expected output (example):

```text
/dev/sdb1: LABEL="data" UUID="9c8d7e6f-0000-0000-0000-000000000000" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="..."
```

Use `blkid` to obtain UUIDs for `/etc/fstab`. Run it as root to see all devices.

## 3. Create a GPT Partition Table

Warning: this destroys all data on the device. Verify the device name with `lsblk` first.

Run as root:

```bash
parted /dev/<DEVICE> mklabel gpt
parted /dev/<DEVICE> mkpart primary ext4 1MiB 100%
parted /dev/<DEVICE> print
```

**Options:**
- `mklabel gpt`: Writes a new GPT partition table.
- `mkpart primary ext4 1MiB 100%`: Creates one partition from 1 MiB to the end of the device. The 1 MiB offset keeps the partition aligned.
- `print`: Displays the resulting layout.

Reload the partition table in the kernel:

```bash
partprobe /dev/<DEVICE>
udevadm settle
```

Verify the result:

```bash
lsblk /dev/<DEVICE>
```

Note on partition names: SATA disks use `<DEVICE>1` (for example `/dev/sdb` → `/dev/sdb1`), while NVMe and loop devices insert `p` (for example `/dev/nvme0n1` → `/dev/nvme0n1p1`).

## 4. Create a Filesystem

For ext4 (default on Debian-based systems):

```bash
mkfs.ext4 -L <LABEL> /dev/<DEVICE>1
```

For XFS (default on RHEL-based systems):

```bash
mkfs.xfs -L <LABEL> /dev/<DEVICE>1
```

**Options:**
- `-L <LABEL>`: Sets a filesystem label.
- `mkfs.xfs -f`: Required to overwrite an existing XFS signature.

`mkfs` destroys existing data on the target. Double-check the device before running it. See Part 1 and Part 2 for distribution-specific choices.

## 5. Mount and Unmount

Create the mount point and mount the filesystem:

```bash
mkdir -p <MOUNT_POINT>
mount /dev/<DEVICE>1 <MOUNT_POINT>
findmnt <MOUNT_POINT>
```

Unmount it:

```bash
umount <MOUNT_POINT>
```

If `umount` reports `target is busy`, processes are still using the filesystem. Identify them:

```bash
fuser -vm <MOUNT_POINT>
```

Close the listed processes before unmounting. Use `umount -l` only when a lazy unmount is acceptable.

## 6. Persist the Mount in /etc/fstab

Register the filesystem by UUID. Example entry in `/etc/fstab`:

```text
UUID=<UUID>  <MOUNT_POINT>  ext4  defaults  0  2
```

Obtain the UUID from `blkid /dev/<DEVICE>1` or `lsblk -f`.

Test the configuration without rebooting:

```bash
mount -a
```

Expected output: none. Fix any error before the next boot. Verify the result:

```bash
findmnt <MOUNT_POINT>
```

## 7. Add Swap

Create swap on a partition or logical volume:

```bash
mkswap /dev/<DEVICE>2
swapon /dev/<DEVICE>2
swapon --show
```

Expected output (example):

```text
NAME      TYPE      SIZE USED PRIO
/dev/sdb2 partition   4G   0B   -2
```

Persist it in `/etc/fstab`:

```text
UUID=<SWAP_UUID>  none  swap  sw  0  0
```

Deactivate swap:

```bash
swapoff /dev/<DEVICE>2
```

**Options:**
- `mkswap`: Writes a swap signature. Destroys existing data on the target.
- `swapon`: Activates the swap area.
- `swapoff`: Deactivates swap. Requires enough free RAM to move swapped pages back.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install the Tools

Run as root:

```bash
apt update
apt install parted e2fsprogs
```

## 2. Use ext4 as the Default Filesystem

Format the partition:

```bash
mkfs.ext4 -L <LABEL> /dev/<DEVICE>1
```

Check the filesystem while unmounted:

```bash
e2fsck -f /dev/<DEVICE>1
```

## 3. Grow the Filesystem Online

After extending the underlying partition or volume, grow ext4 while mounted:

```bash
resize2fs /dev/<DEVICE>1
```

ext4 can also be shrunk offline after unmounting and running `e2fsck -f`, but shrinking is risky and rarely needed. Take a backup first.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install the Tools

Run as root:

```bash
dnf install parted xfsprogs
```

## 2. Use XFS as the Default Filesystem

Format the partition:

```bash
mkfs.xfs -L <LABEL> /dev/<DEVICE>1
```

Check the filesystem while unmounted:

```bash
xfs_repair -n /dev/<DEVICE>1
```

## 3. Grow the Filesystem Online

After extending the underlying partition or volume, grow XFS from its mount point while mounted:

```bash
xfs_growfs <MOUNT_POINT>
```

XFS cannot be shrunk. Size the filesystem conservatively and grow it later. Inspect the layout with:

```bash
xfs_info <MOUNT_POINT>
```

---

# Part 3: Important Notes

## fstab Fields

| Field | Meaning | Example |
|---|---|---|
| 1: Device | Block device, UUID, or label | `UUID=<UUID>` |
| 2: Mount point | Directory where the filesystem is mounted | `<MOUNT_POINT>` |
| 3: Type | Filesystem type | `ext4`, `xfs`, `swap` |
| 4: Options | Mount options | `defaults`, `noexec,nosuid` |
| 5: Dump | Dump backup flag, usually `0` | `0` |
| 6: Pass | fsck order: `1` root, `2` other, `0` to skip | `2` |

## Filesystem Comparison

| Property | ext4 | XFS |
|---|---|---|
| Default on | Debian, Ubuntu | RHEL, Rocky Linux, AlmaLinux |
| Shrink online | No | No |
| Shrink offline | Yes | No |
| Grow online | `resize2fs` | `xfs_growfs <MOUNT_POINT>` |
| Check tool | `e2fsck` | `xfs_repair` |
| Metadata behavior | Mature and conservative | Scales better on large volumes and parallel I/O |

## Security Guidelines

- Double-check the device name with `lsblk` before running `mkfs`. `mkfs` on the wrong device destroys data irrecoverably.
- Prefer UUIDs in `/etc/fstab`. Device names such as `/dev/sdb1` can change between boots.
- Never edit `/etc/fstab` without testing the result with `mount -a`. A broken entry can leave the system in emergency mode at boot.
- Run `fsck` or `xfs_repair` only on unmounted filesystems unless the tool documents online checks.
- Use `noexec,nosuid,nodev` on mounts holding untrusted or shared data.
- Do not store sensitive data on world-readable mount points; restrict directory permissions after mounting.
- Keep swap on a device separate from the root filesystem where possible, and remember that swap contents are unencrypted by default.
- In scripts, refuse to proceed when the target already contains a filesystem signature unless overwrite is intended.
