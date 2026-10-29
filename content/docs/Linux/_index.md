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
4. [system-information](system-information/) = `/etc/os-release`, `uname`, `hostnamectl`, distro identification
5. [sudo](sudo/) = `visudo`, `/etc/sudoers`, `sudoers.d`, NOPASSWD
6. [ssh](ssh/) = key generation, `authorized_keys`, `sshd_config` hardening
7. [systemd](systemd/) = `systemctl`, unit files, `journalctl`
8. [process-management](process-management/) = `ps`, `top`, `kill`, signals, zombies
9. [process-internals](process-internals/) = programs as files, fork/exec, child processes, namespaces
10. [cron-timers](cron-timers/) = `crontab` syntax, `[Timer]` units
11. [networking-basics](networking-basics/) = `ip`, `nmcli`, netplan, interfaces
12. [dns](dns/) = `/etc/hosts`, `resolvectl`, `dig`
13. [firewall](firewall/) = port rules, zones, persistence
14. [network-troubleshooting](network-troubleshooting/) = `ss`, `curl`, `tcpdump`, layered diagnosis
15. [filesystem-hierarchy](filesystem-hierarchy/) = what lives where on a Linux system
16. [disk-management](disk-management/) = `lsblk`, `parted`, `mkfs`, `/etc/fstab`, swap
17. [lvm](lvm/) = logical volumes, extending volumes and filesystems
18. [disk-troubleshooting](disk-troubleshooting/) = `df`, `du`, inodes, deleted-but-open files
19. [log-management](log-management/) = `/var/log`, `journalctl` queries, `logrotate`
20. [selinux-apparmor](selinux-apparmor/) = MAC, contexts, profiles, diagnosing denials
21. [security-hardening](security-hardening/) = baseline hardening, brute-force protection
22. [package-management](package-management/) = install, upgrade, repos, holds
23. [performance-triage](performance-triage/) = load average, CPU vs I/O wait, memory, disk I/O
24. [boot-process](boot-process/) = GRUB → initramfs → systemd targets
