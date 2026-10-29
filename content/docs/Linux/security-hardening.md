---
title: "security-hardening"
weight: 190
---
**Scope:** Debian-based and RHEL-based Linux server hardening  
**Topics:** Baseline hardening, fail2ban, automatic security updates, listening ports, login auditing.

Server hardening reduces the attack surface before an incident occurs: keep the system patched, expose only required services, restrict administrative access, and monitor authentication attempts. Fail2ban complements these measures by banning hosts that repeatedly fail authentication. Treat the steps below as a baseline and adjust them to the role of each server.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install fail2ban

Run as root:

```bash
apt update
apt install fail2ban
```

## 2. Configure the sshd Jail

Create or edit `/etc/fail2ban/jail.local`. Do not edit `jail.conf`; it is overwritten on package upgrades.

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 3
```

**Options:**
- `bantime`: Duration of a ban. `1h` bans an address for one hour; `-1` bans permanently.
- `findtime`: Window in which failures are counted.
- `maxretry`: Number of failures within `findtime` that trigger a ban.
- `ignoreip`: Addresses that are never banned. Keep management hosts here.
- `logpath`: Authentication log monitored by the jail. Debian-based systems use `/var/log/auth.log`.

## 3. Enable and Start fail2ban

```bash
systemctl enable --now fail2ban
systemctl status fail2ban
```

Confirm that the service is active and running.

## 4. Verify the Jail

Inspect the active jails:

```bash
fail2ban-client status
```

Inspect the sshd jail:

```bash
fail2ban-client status sshd
```

Expected output:

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 1
|  `- Total failed:     6
`- Actions
   |- Currently banned: 1
   `- Banned IP list:   203.0.113.10
```

Unban an address if necessary:

```bash
fail2ban-client set sshd unbanip <IP_ADDRESS>
```

## 5. Enable Unattended Security Updates

Install the automatic upgrade tool:

```bash
apt install unattended-upgrades
```

Enable it interactively and select "Yes" when prompted:

```bash
dpkg-reconfigure -plow unattended-upgrades
```

Inspect the generated configuration:

```bash
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Example:

```text
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Verify that upgrades were applied by inspecting `/var/log/unattended-upgrades/unattended-upgrades.log`.

## 6. Check Listening Ports

List listening TCP and UDP sockets with the owning process:

```bash
ss -tulpn
```

Expected output:

```text
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=812,fd=3))
```

Every unexpected listener is an unnecessary exposure. Disable the corresponding service:

```bash
systemctl disable --now <SERVICE>
```

## 7. Review Login History

Inspect recent logins:

```bash
last
lastlog
```

Inspect failed login attempts:

```bash
lastb | head -n 20
```

Inspect SSH authentication events:

```bash
journalctl -u ssh --since today
```

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install fail2ban from EPEL

Fail2ban is not in the default repositories. Enable EPEL first:

```bash
dnf install epel-release
dnf install fail2ban
```

On RHEL itself, install the EPEL release package directly with `dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm`.

## 2. Configure the sshd Jail

Create or edit `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8 ::1

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/secure
maxretry = 3
```

**Options:**
- `bantime`: Duration of a ban. `1h` bans an address for one hour; `-1` bans permanently.
- `findtime`: Window in which failures are counted.
- `maxretry`: Number of failures within `findtime` that trigger a ban.
- `ignoreip`: Addresses that are never banned. Keep management hosts here.
- `logpath`: Authentication log monitored by the jail. RHEL-based systems use `/var/log/secure`.

## 3. Enable and Start fail2ban

If firewalld is active, fail2ban inserts bans through it automatically:

```bash
systemctl enable --now firewalld
systemctl enable --now fail2ban
systemctl status fail2ban
```

## 4. Verify the Jail

Inspect the active jails:

```bash
fail2ban-client status
```

Inspect the sshd jail:

```bash
fail2ban-client status sshd
```

Expected output:

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  `- Total failed:     3
`- Actions
   |- Currently banned: 1
   `- Banned IP list:   203.0.113.10
```

Unban an address if necessary:

```bash
fail2ban-client set sshd unbanip <IP_ADDRESS>
```

## 5. Enable Automatic Updates

Install the automatic update tool:

```bash
dnf install dnf-automatic
```

Edit `/etc/dnf/automatic.conf` and set:

```ini
[commands]
upgrade_type = security
apply_updates = yes
```

Enable the timer:

```bash
systemctl enable --now dnf-automatic.timer
systemctl status dnf-automatic.timer
```

Verify that upgrades were applied by inspecting `/var/log/dnf.rpm.log`.

## 6. Check Listening Ports

List listening TCP and UDP sockets with the owning process:

```bash
ss -tulpn
```

Disable any unexpected listener:

```bash
systemctl disable --now <SERVICE>
```

## 7. Review Login History

Inspect recent logins:

```bash
last
lastlog
```

Inspect failed login attempts:

```bash
lastb | head -n 20
```

Inspect SSH authentication events:

```bash
journalctl -u sshd --since today
```

---

# Part 3: Important Notes

## Hardening Checklist

| Area | Action | Command |
|---|---|---|
| Brute-force protection | Ban repeated authentication failures | `fail2ban-client status sshd` |
| Patching | Apply security updates automatically | `dpkg-reconfigure -plow unattended-upgrades`, `systemctl enable --now dnf-automatic.timer` |
| Attack surface | Disable services that are not required | `systemctl disable --now <SERVICE>` |
| Open ports | Audit listening sockets | `ss -tulpn` |
| SSH access | Restrict to key-based authentication | `PasswordAuthentication no` in `/etc/ssh/sshd_config` |
| Privilege escalation | Use a regular account with sudo | `sudo -l` |
| Logins | Review successful and failed logins | `last`, `lastb`, `lastlog` |
| Time | Keep time synchronized for log correlation | `timedatectl`, `chronyc tracking` |
| Firewall | Allow only required inbound traffic | `nft list ruleset`, `firewall-cmd --list-all` |
| Fail2ban policy | Review jails after every configuration change | `fail2ban-client status` |

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Install fail2ban | `apt install fail2ban` | `dnf install epel-release && dnf install fail2ban` |
| Jail configuration | `/etc/fail2ban/jail.local` | `/etc/fail2ban/jail.local` |
| SSH auth log | `/var/log/auth.log` | `/var/log/secure` |
| Enable service | `systemctl enable --now fail2ban` | `systemctl enable --now fail2ban` |
| Automatic updates | `unattended-upgrades` | `dnf-automatic` |
| Listening sockets | `ss -tulpn` | `ss -tulpn` |
| Failed logins | `lastb` | `lastb` |

## Security Guidelines

- Apply the principle of least privilege: grant only the access each account and service requires.
- Use key-based SSH authentication and disable password logins as a complement to fail2ban, not as a replacement for it.
- Keep `ignoreip` limited to trusted management hosts so that you are never locked out by your own jail.
- Start with conservative values such as `maxretry = 3` in the `sshd` jail and raise `bantime` only after observing real traffic.
- Review `fail2ban-client status` after every change to `jail.local` and restart the service to apply it.
- Do not edit `jail.conf`; local overrides belong in `jail.local`.
- Verify automatic updates by reading the update logs instead of assuming they work.
- Audit `ss -tulpn` after installing new software; packages frequently enable services automatically.
- Keep authentication logs and review `lastb` regularly; fail2ban only reacts to patterns it can see.
- Restrict physical and console access to servers that expose administrative interfaces.