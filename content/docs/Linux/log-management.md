---
title: "log-management"
weight: 170
---
**Scope:** Log sources, locations, querying, and rotation on Debian-based and RHEL-based distributions  
**Topics:** rsyslog, /var/log layout, journald, journalctl, logrotate.

## 1. Understand the Log Sources

Two logging systems handle messages on current distributions:

- systemd-journald: Collects messages from the kernel, services, and stdout/stderr of units. Stores them in the journal, which is volatile in `/run/log/journal` unless persistent storage is configured.
- rsyslog: Traditional syslog daemon. Reads `/etc/rsyslog.conf` and `/etc/rsyslog.d/*.conf` and writes text files under `/var/log`. On RHEL-based systems it imports messages from the journal through the `imjournal` module.

Rules in rsyslog use `<FACILITY>.<PRIORITY>` selectors. Example configuration in `/etc/rsyslog.d/50-custom.conf`:

```text
auth,authpriv.*                 /var/log/auth.log
kern.*                          /var/log/kern.log
*.*;auth,authpriv.none          -/var/log/syslog
```

**Options:**
- `auth`, `kern`: Facilities (message sources).
- `*`, `none`: Priorities from `emerg` down to `debug`; `none` excludes a facility.
- `-` before the path: Enables asynchronous writes for high-volume logs.

## 2. Inspect the /var/log Layout

| Path | Contents |
|---|---|
| `/var/log/syslog` | General system messages (Debian-based) |
| `/var/log/messages` | General system messages (RHEL-based) |
| `/var/log/auth.log` | Authentication and sudo events (Debian-based) |
| `/var/log/secure` | Authentication and sudo events (RHEL-based) |
| `/var/log/kern.log` | Kernel messages (Debian-based) |
| `/var/log/cron` | Cron scheduler activity |
| `/var/log/audit/audit.log` | Audit subsystem and SELinux events (RHEL-based) |
| `/var/log/journal/` | Persistent journal storage |
| `/var/log/dpkg.log`, `/var/log/dnf.log` | Package transactions |

Binary accounting files also live here: `/var/log/wtmp` (successful logins, read with `last`) and `/var/log/btmp` (failed logins, read with `lastb`).

List files and sizes:

```bash
ls -lh /var/log
du -xh -d 1 /var/log | sort -h
```

## 3. Query the Journal with journalctl

Inspect the usage:

```bash
journalctl -u <SERVICE>
journalctl --since "2026-01-01 10:00:00" --until "2026-01-01 12:00:00"
journalctl -p err
journalctl -b
journalctl -f
```

**Options:**
- `-u <SERVICE>`: Shows messages for one systemd unit.
- `--since`, `--until`: Time window in `YYYY-MM-DD HH:MM:SS` form. Relative values such as `yesterday`, `today`, or `-1h` are also accepted.
- `-p err`: Filters by priority (`emerg`, `alert`, `crit`, `err`, `warning`, `notice`, `info`, `debug`) and includes all higher priorities.
- `-b`: Current boot. Use `-b -1` for the previous boot.
- `-f`: Follows new messages like `tail -f`.
- `-n 50`: Shows the last 50 entries.
- `--no-pager`: Plain output for scripts.

Expected output (example):

```text
Jan 29 10:12:33 host sshd[1234]: Accepted publickey for deploy from 10.0.0.5 port 51234 ssh2
```

## 4. Rotate Logs with logrotate

logrotate ages and compresses log files on schedule through `logrotate.timer` or a cron job. Distribution defaults live in `/etc/logrotate.conf`; per-application drop-ins go in `/etc/logrotate.d/`.

Example configuration in `/etc/logrotate.d/<APP>`:

```text
/var/log/<APP>/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        systemctl reload <APP> > /dev/null 2>&1 || true
    endscript
}
```

**Options:**
- `daily`: Rotation frequency (`daily`, `weekly`, `monthly`).
- `rotate 14`: Number of rotated archives to keep.
- `compress`, `delaycompress`: Compress rotated logs, delaying compression by one cycle.
- `missingok`, `notifempty`: Do not fail on missing files and skip empty logs.
- `create 0640 root adm`: Mode and ownership of the newly created log file.
- `sharedscripts`, `postrotate`: Run the script block once after rotation, typically to make the service reopen its log files.

Test the configuration without changing files:

```bash
logrotate -d /etc/logrotate.conf
```

Force a rotation now:

```bash
logrotate -f /etc/logrotate.conf
```

Force a single drop-in:

```bash
logrotate -f /etc/logrotate.d/<APP>
```

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Read Debian-Specific Log Files

```bash
tail -n 100 /var/log/syslog
tail -n 100 /var/log/auth.log
tail -n 100 /var/log/kern.log
```

- `/var/log/syslog`: General system messages.
- `/var/log/auth.log`: Authentication and sudo events. Monitor for repeated failures.
- `/var/log/kern.log`: Kernel messages, hardware errors, and filesystem warnings.

These files are readable by the `adm` group. Grant read access to trusted administrators without giving them a root shell:

```bash
usermod -aG adm <USERNAME>
```

## 2. Read Package and Authentication History

```bash
grep " install " /var/log/dpkg.log
zgrep -h "Failed password" /var/log/auth.log*
```

**Options:**
- `zgrep`: Searches compressed (`.gz`) rotated logs as well as plain files.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Read RHEL-Specific Log Files

```bash
tail -n 100 /var/log/messages
tail -n 100 /var/log/secure
```

- `/var/log/messages`: General system messages written by rsyslog.
- `/var/log/secure`: Authentication and sudo events.
- `/var/log/audit/audit.log`: Audit subsystem and SELinux denial events.

## 2. Enable Persistent Journal Storage

The journal is volatile by default and loses its history at every reboot. Enable persistent storage and cap its size in `/etc/systemd/journald.conf`:

```ini
[Journal]
Storage=persistent
SystemMaxUse=1G
```

Apply the configuration:

```bash
mkdir -p /var/log/journal
systemctl restart systemd-journald
```

Messages now survive reboots and can be queried across boots:

```bash
journalctl -b -1
```

---

# Part 3: Important Notes

## Log File Comparison

| Topic | Debian-based | RHEL-based |
|---|---|---|
| General messages | `/var/log/syslog` | `/var/log/messages` |
| Authentication | `/var/log/auth.log` | `/var/log/secure` |
| Kernel messages | `/var/log/kern.log` | `/var/log/messages`, `journalctl -k` |
| Package log | `/var/log/dpkg.log` | `/var/log/dnf.log`, `/var/log/yum.log` |
| Default ownership | `root:adm`, mode `0640` | `root:root`, mode `0600` |
| Log reader group | `adm` | `root` |
| Journald persistence | `/var/log/journal` when enabled | `/var/log/journal` when enabled |

## Security Guidelines

- Centralize logs on a remote host or log platform. A compromised host cannot be trusted to report its own logs.
- Restrict log permissions: `0640 root:adm` on Debian-based and `0600 root:root` on RHEL-based systems. Never make logs world-readable.
- Do not rotate live logs manually with `mv` or `cp`. Use logrotate so services can reopen their files consistently.
- Never log credentials, tokens, or private keys. Redact secrets at the application level before messages reach rsyslog or journald.
- Retain authentication and audit logs (`auth.log`, `secure`, `audit.log`) for a defined period that covers your incident response window.
- Back up `/etc/rsyslog.conf`, `/etc/rsyslog.d`, and `/etc/logrotate.d` together with the rest of `/etc`.
- Protect `/var/log/journal`; it is a common target for evidence removal during intrusions.
- Record the `journalctl` queries used during an incident, since retained history changes with vacuum and `SystemMaxUse` settings.
