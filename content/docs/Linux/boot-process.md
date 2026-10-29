---
title: "boot-process"
weight: 220
---
**Scope:** Boot chain, systemd targets, and GRUB configuration on Debian-based and RHEL-based systems  
**Topics:** UEFI and GRUB, kernel and initramfs, systemd targets, boot timing, kernel parameters, boot recovery.

A Linux server boots through a fixed chain: the firmware (UEFI or BIOS) starts the GRUB 2 bootloader, GRUB loads the kernel and initramfs, the kernel hands control to systemd, systemd reaches the default target, and a login prompt appears. Each stage fails in a distinct way, so knowing the chain narrows a boot problem quickly. The steps below cover diagnosis first, then GRUB configuration per distribution family, then recovery.

## 1. Inspect Boot Timing

Measure how long the last boot took:

```bash
systemd-analyze time
```

Expected output:

```text
Startup finished in 2.103s (firmware) + 1.842s (loader) + 1.201s (kernel) + 8.412s (userspace) = 13.558s
graphical.target reached after 8.390s in userspace.
```

Rank the slowest services:

```bash
systemd-analyze blame
```

Inspect the critical path to a target:

```bash
systemd-analyze critical-chain
```

**Options:**
- `time`: Shows the duration of each boot stage.
- `blame`: Lists services sorted by startup time.
- `critical-chain`: Shows the dependency chain that delayed the target.

## 2. Select a systemd Target

Inspect the default target:

```bash
systemctl get-default
```

Expected output:

```text
multi-user.target
```

Change the default target:

```bash
systemctl set-default multi-user.target
```

List active targets:

```bash
systemctl list-units --type target --state active
```

| Target | Purpose |
|---|---|
| `multi-user.target` | Multi-user text-mode system with networking. Server default. |
| `graphical.target` | Multi-user system plus graphical login. Workstation default. |
| `rescue.target` | Single-user mode with the root filesystem mounted read-write. |
| `emergency.target` | Minimal shell with only the root filesystem mounted read-only. |

Switch to a target for the current boot only:

```bash
systemctl isolate rescue.target
```

## 3. Recover a Broken Boot

Interrupt the boot at the GRUB menu and press `e` to edit the selected entry. Find the line starting with `linux` and append a recovery parameter at the end of it.

**Options:**
- `systemd.unit=rescue.target`: Boots into rescue mode with the root filesystem read-write.
- `systemd.unit=emergency.target`: Boots the smallest possible environment.
- `init=/bin/bash`: Bypasses systemd entirely and starts a root shell.
- `rd.break` (RHEL-based): Stops inside the initramfs before the real root is mounted.

Boot the edited entry with `Ctrl+x` (BIOS) or `F10` (UEFI). Changes made this way affect one boot only. When `rd.break` is used, remount the real root filesystem and reset the root password:

```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
exit
reboot
```

Inspect the previous boot after a failure or an unattended reboot:

```bash
journalctl -b -1
journalctl -b -1 -p err
```

**Options:**
- `-b -1`: Shows the log of the boot before the current one.
- `-b`: Shows the log of the current boot.
- `-p err`: Filters messages of priority error and higher.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Inspect the GRUB Configuration

Inspect the defaults file:

```bash
cat /etc/default/grub
```

Example:

```text
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX=""
```

## 2. Set Kernel Parameters

Edit `/etc/default/grub`. Parameters that apply to every boot belong in `GRUB_CMDLINE_LINUX`:

```text
GRUB_CMDLINE_LINUX="net.ifnames=0 console=ttyS0,115200n8"
```

**Options:**
- `GRUB_CMDLINE_LINUX`: Applied to all entries, including recovery entries.
- `GRUB_CMDLINE_LINUX_DEFAULT`: Applied to the default entries only. Removing `quiet` shows full boot messages.
- `GRUB_DEFAULT=0`: Boots the first menu entry. Use a menu index or a title string to select another entry.
- `GRUB_TIMEOUT`: Seconds the menu is displayed before the default entry boots.

## 3. Regenerate the GRUB Configuration

Run after every change to `/etc/default/grub`:

```bash
update-grub
```

Expected output:

```text
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-6.1.0-18-amd64
Found initrd image: /boot/initrd.img-6.1.0-18-amd64
done
```

`update-grub` is a wrapper for `grub-mkconfig -o /boot/grub/grub.cfg`. Never edit `/boot/grub/grub.cfg` by hand.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Inspect the GRUB Configuration

Inspect the defaults file:

```bash
cat /etc/default/grub
```

Example:

```text
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_CMDLINE_LINUX="crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
GRUB_ENABLE_BLSCFG=true
```

## 2. Set the Default Entry and Kernel Parameters

With `GRUB_DEFAULT=saved`, the default entry is stored in the GRUB environment block. Set it explicitly:

```bash
grub2-set-default <MENU_INDEX_OR_TITLE>
grub2-editenv list
```

Alternatively, set `GRUB_DEFAULT=0` in `/etc/default/grub` to always boot the first entry.

Edit `GRUB_CMDLINE_LINUX` to add parameters for every boot. Remove `rhgb quiet` on troubleshooting systems to see full boot messages:

```text
GRUB_CMDLINE_LINUX="crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M"
```

## 3. Regenerate the GRUB Configuration

Run after every change to `/etc/default/grub`:

```bash
grub2-mkconfig -o /boot/grub2/grub.cfg
```

On RHEL 9 and its rebuilds, `/boot/grub2/grub.cfg` is the authoritative configuration for BIOS and UEFI systems. Older layouts keep the file on the EFI system partition at `/boot/efi/EFI/<VENDOR>/grub.cfg`; check the path with:

```bash
find /boot -name grub.cfg
```

Kernel entries are managed as BLS entries in `/boot/loader/entries/`. Never edit `grub.cfg` or the entry files by hand.

---

# Part 3: Important Notes

## Boot Stage Reference

| Stage | Component | Failure symptom |
|---|---|---|
| 1. Firmware | UEFI or BIOS | No bootloader menu; "no bootable device found" |
| 2. Bootloader | GRUB 2 | `grub>` rescue prompt; "error: no such partition" |
| 3. Kernel and initramfs | Kernel, initramfs | Kernel panic; dropped to an initramfs shell |
| 4. Init system | systemd | Stuck before login; failed units in `systemctl --failed` |
| 5. Target | systemd target | Wrong target reached; no network; no graphical login |
| 6. Login | getty, display manager | Login prompt missing; authentication failures |

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| GRUB defaults | `/etc/default/grub` | `/etc/default/grub` |
| Regenerate configuration | `update-grub` | `grub2-mkconfig -o /boot/grub2/grub.cfg` |
| Set default entry | `GRUB_DEFAULT=<INDEX>` | `grub2-set-default <INDEX_OR_TITLE>` |
| Boot timing | `systemd-analyze time` | `systemd-analyze time` |
| Default target | `systemctl get-default` | `systemctl get-default` |
| Previous boot log | `journalctl -b -1` | `journalctl -b -1` |
| Recovery parameter | `systemd.unit=rescue.target` | `rd.break` |

## Security Guidelines

- Keep console or out-of-band access open before changing GRUB; a broken configuration can leave the system unreachable.
- Always regenerate `grub.cfg` with the distribution's own tool: `update-grub` on Debian-based systems, `grub2-mkconfig` on RHEL-based systems.
- Never edit `grub.cfg` directly; the changes are lost on the next regeneration and are not validated.
- Protect the GRUB menu with a password on physically accessible systems so that kernel parameters cannot be edited at boot.
- Remember that `e` at the GRUB menu and `rd.break` grant root access; lock down console access accordingly.
- Keep at least one older kernel installed and bootable for rollback after a bad kernel or driver update.
- Do not remove recovery entries or the initramfs images; they are the fastest path back into a broken system.
- Restrict write access to `/boot`, `/etc/default/grub`, and `/boot/loader/entries/` to root.
- After changing the default target or kernel parameters, reboot once and verify with `systemctl get-default` and `systemd-analyze time`.
- Use `journalctl -b -1` to investigate unexpected reboots before cleaning old journals.