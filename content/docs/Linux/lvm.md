---
title: "lvm"
weight: 150
---
**Scope:** Logical Volume Management on Debian-based and RHEL-based distributions  
**Topics:** PV, VG, LV, pvcreate, vgcreate, lvcreate, lvextend, resize2fs, xfs_growfs, cloud volume extension.

## 1. Understand the LVM Model

LVM places an abstraction layer between physical disks and filesystems:

- Physical Volume (PV): A disk or partition registered with LVM.
- Volume Group (VG): A storage pool built from one or more PVs.
- Logical Volume (LV): A flexible volume carved from a VG, formatted and mounted like a partition.

An LV appears as `/dev/<VG_NAME>/<LV_NAME>`. Unlike partitions, LVs can be extended while mounted.

## 2. Create the Physical Volume

Warning: this destroys existing data on the target. Run as root:

```bash
pvcreate /dev/<DEVICE>1
```

Verify the result:

```bash
pvs
```

Expected output (example):

```text
PV         VG   Fmt  Attr PSize   PFree
/dev/sdb1  appvg lvm2 a--  <50.00g <50.00g
```

**Options:**
- `/dev/<DEVICE>1`: The partition to register. The whole disk (`/dev/<DEVICE>`) can also serve as a PV.

## 3. Create the Volume Group

```bash
vgcreate <VG_NAME> /dev/<DEVICE>1
```

Verify the result:

```bash
vgs
```

Expected output (example):

```text
VG    #PV #LV #SN Attr   VSize   VFree
appvg   1   0   0 wz--n- <50.00g <50.00g
```

Add another disk to the pool later with `vgextend <VG_NAME> /dev/<DEVICE>2`.

## 4. Create the Logical Volume

Allocate a fixed size:

```bash
lvcreate -L 10G -n <LV_NAME> <VG_NAME>
```

Or consume all remaining space:

```bash
lvcreate -l 100%FREE -n <LV_NAME> <VG_NAME>
```

Verify the result:

```bash
lvs
```

Expected output (example):

```text
LV       VG    Attr       LSize   Pool Origin Data%  Meta%  Log Cpy%Sync Convert
datalv   appvg -wi-a-----  10.00g
```

**Options:**
- `-L 10G`: Absolute size. Use `-L +5G` to add to an existing LV.
- `-l 100%FREE`: Size in extents, here all free space in the VG.
- `-n <LV_NAME>`: Name of the logical volume.

## 5. Create the Filesystem and Mount

Format the LV. Use `mkfs.ext4` on Debian-based and `mkfs.xfs` on RHEL-based systems:

```bash
mkfs.ext4 /dev/<VG_NAME>/<LV_NAME>
mkdir -p <MOUNT_POINT>
mount /dev/<VG_NAME>/<LV_NAME> <MOUNT_POINT>
findmnt <MOUNT_POINT>
```

Persist the mount in `/etc/fstab` by UUID:

```text
UUID=<UUID>  <MOUNT_POINT>  ext4  defaults  0  2
```

Obtain the UUID with `blkid /dev/<VG_NAME>/<LV_NAME>` and test the entry with `mount -a`.

## 6. Verify the LVM Stack

Inspect every layer:

```bash
pvs
vgs
lvs
```

Inspect the device tree:

```bash
lsblk
```

Expected output (example):

```text
sdb
└─sdb1                8:17   0   50G  0 lvm
  └─appvg-datalv    253:0    0   10G  0 /srv/data
```

## 7. Extend a Logical Volume

Grow the VG first when it has no free space:

```bash
vgextend <VG_NAME> /dev/<DEVICE>2
```

Extend the LV by a fixed amount or to use all remaining space:

```bash
lvextend -L +10G /dev/<VG_NAME>/<LV_NAME>
lvextend -l +100%FREE /dev/<VG_NAME>/<LV_NAME>
```

Let LVM resize the filesystem in the same step:

```bash
lvextend -r -l +100%FREE /dev/<VG_NAME>/<LV_NAME>
```

**Options:**
- `-L +10G`: Adds 10 GiB to the current size. Without `+` the value is the new absolute size.
- `-l +100%FREE`: Adds all remaining VG space.
- `-r`, `--resizefs`: Grows the filesystem after the LV using the appropriate tool (resize2fs or xfs_growfs).

When not using `-r`, the filesystem must be grown separately. Use the steps in Part 1 for ext4 or Part 2 for XFS.

## 8. Extend a Cloud-Attached Volume

Scenario: the LV sits on a cloud disk that was enlarged in the provider console, for example from 50 GiB to 100 GiB. Extend each layer in order.

Resize the partition first if the PV is a partition rather than the whole disk:

```bash
growpart /dev/<DEVICE> 1
```

**Options:**
- `growpart`: Resizes a partition to fill the device. Provided by `cloud-guest-utils` on Debian-based and `cloud-utils-growpart` on RHEL-based systems.

Resize the PV to the new device size:

```bash
pvresize /dev/<DEVICE>1
```

Confirm that the VG now reports free space:

```bash
vgs
```

Extend the LV and filesystem in one step:

```bash
lvextend -r -l +100%FREE /dev/<VG_NAME>/<LV_NAME>
```

Verify the result:

```bash
df -h <MOUNT_POINT>
```

If the PV uses the whole disk, `growpart` is unnecessary: `pvresize` alone picks up the new device size.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Grow an ext4 Filesystem

After `lvextend` without `-r`, grow ext4 while mounted:

```bash
resize2fs /dev/<VG_NAME>/<LV_NAME>
```

Verify the result:

```bash
df -h <MOUNT_POINT>
```

## 2. Check an ext4 Filesystem

Run on the unmounted LV only:

```bash
e2fsck -f /dev/<VG_NAME>/<LV_NAME>
```

ext4 can be shrunk offline after unmounting, but treat shrinking as a last resort and take a backup first.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Grow an XFS Filesystem

After `lvextend` without `-r`, grow XFS from its mount point while mounted:

```bash
xfs_growfs <MOUNT_POINT>
```

Verify the result:

```bash
df -h <MOUNT_POINT>
```

## 2. Check an XFS Filesystem

Run on the unmounted LV only:

```bash
xfs_repair -n /dev/<VG_NAME>/<LV_NAME>
```

XFS cannot be shrunk. Plan capacity accordingly and prefer extending over pre-allocating large volumes.

---

# Part 3: Important Notes

## LVM Layers

| Layer | Creation command | Purpose | Example |
|---|---|---|---|
| PV | `pvcreate` | Registers a device with LVM | `pvcreate /dev/<DEVICE>1` |
| VG | `vgcreate` | Creates a storage pool | `vgcreate <VG_NAME> /dev/<DEVICE>1` |
| LV | `lvcreate` | Allocates a volume from the pool | `lvcreate -L 10G -n <LV_NAME> <VG_NAME>` |
| Filesystem | `mkfs` | Formats the LV | `mkfs.ext4 /dev/<VG_NAME>/<LV_NAME>` |

## Extension Commands

| Operation | Command |
|---|---|
| Add a disk to a VG | `vgextend <VG_NAME> /dev/<DEVICE>2` |
| Grow a PV after a device resize | `pvresize /dev/<DEVICE>1` |
| Set absolute LV size | `lvextend -L 20G /dev/<VG_NAME>/<LV_NAME>` |
| Add to LV size | `lvextend -L +10G /dev/<VG_NAME>/<LV_NAME>` |
| Use all free VG space | `lvextend -l +100%FREE /dev/<VG_NAME>/<LV_NAME>` |
| Grow LV and filesystem together | `lvextend -r ...` |
| Grow ext4 (Debian-based) | `resize2fs /dev/<VG_NAME>/<LV_NAME>` |
| Grow XFS (RHEL-based) | `xfs_growfs <MOUNT_POINT>` |

## Security Guidelines

- Leave free space in the VG for snapshots and emergency growth. Consuming 100% of the VG removes these options.
- Never shrink an XFS filesystem; no supported tooling exists for it.
- Take a backup before any size reduction, `fsck`, or partition change.
- LVM snapshots consume the same VG space and can fill up silently. Monitor them with `lvs` and do not treat snapshots as backups.
- Run `pvcreate`, `vgcreate`, and `mkfs` only on devices verified with `lsblk` and `blkid`.
- Do not run `fsck`, `resize2fs` (shrinking), or `xfs_repair` on mounted filesystems.
- Do not remove a PV with `vgreduce` while it still holds extents in use. Check with `pvdisplay` first.
- Use descriptive VG and LV names such as `appvg-datalv` so `df` and `lvs` output stays readable during incidents.
- When scripting, treat a non-empty `vgs` `VFree` column as the precondition for `lvextend`.
