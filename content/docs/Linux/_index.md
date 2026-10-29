---
title: "Linux"
weight: 2
bookCollapseSection: true
---
Linux is _the_ foundation of nearly every DevOps workflow. From homelab servers to production clusters, everything runs on it. I use Linux to:

- Manage users, groups and permissions
- Operate services and processes
- Configure networking and storage
- Automate administration tasks with the shell
- And much more

## Linux konular

1. [linux-user-management](linux-user-management/) = user creation, system users, sudo privileges on Debian- and RHEL-based systems
2. [group-management](group-management/) = group creation, primary vs supplementary groups, `/etc/group`
3. [file-permissions](file-permissions/) = `chmod`, `chown`, `umask`, SUID/SGID/sticky bit, ACLs
4. [sudo](sudo/) = `visudo`, `/etc/sudoers`, `sudoers.d`, NOPASSWD
5. [ssh](ssh/) = key generation, `authorized_keys`, `sshd_config` hardening
6. [systemd](systemd/) = `systemctl`, unit files, `journalctl`
7. [process-management](process-management/) = `ps`, `top`, `kill`, signals, zombies
8. [cron-timers](cron-timers/) = `crontab` syntax, `[Timer]` units
9. [networking-basics](networking-basics/) = `ip`, `nmcli`, netplan, interfaces
10. [dns](dns/) = `/etc/hosts`, `resolvectl`, `dig`
11. [firewall](firewall/) = port rules, zones, persistence
12. [network-troubleshooting](network-troubleshooting/) = `ss`, `curl`, `tcpdump`, layered diagnosis
13. [filesystem-hierarchy](filesystem-hierarchy/) = what lives where on a Linux system
14. [disk-management](disk-management/) = `lsblk`, `parted`, `mkfs`, `/etc/fstab`, swap
15. [lvm](lvm/) = logical volumes, extending volumes and filesystems
16. [disk-troubleshooting](disk-troubleshooting/) = `df`, `du`, inodes, deleted-but-open files
17. [log-management](log-management/) = `/var/log`, `journalctl` queries, `logrotate`
18. [selinux-apparmor](selinux-apparmor/) = MAC, contexts, profiles, diagnosing denials
19. [security-hardening](security-hardening/) = baseline hardening, brute-force protection
20. [package-management](package-management/) = install, upgrade, repos, holds
21. [performance-triage](performance-triage/) = load average, CPU vs I/O wait, memory, disk I/O
22. [boot-process](boot-process/) = GRUB → initramfs → systemd targets
