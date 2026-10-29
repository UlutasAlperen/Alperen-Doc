---
title: "cron-timers"
weight: 80
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Crontab syntax, system-wide cron files, cron access control, cron logging, and systemd timers.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install Cron

Run as root:

```bash
apt update
apt install cron
systemctl enable --now cron
```

The service is named `cron`. Verify:

```bash
systemctl status cron
```

## 2. Edit the User Crontab

Open the crontab editor, list entries, or remove the crontab:

```bash
crontab -e
crontab -l
crontab -r
```

**Options:**
- `-e`: Opens the crontab in the editor set by `EDITOR`, defaulting to `vim`.
- `-l`: Lists the current user's crontab.
- `-r`: Removes the current user's crontab without a confirmation prompt. Use `crontab -ri` to confirm first.
- `-u <USERNAME>`: Operates on another user's crontab. Requires root.

## 3. Schedule a Job

Cron syntax uses five time fields followed by the command:

```text
* * * * * <COMMAND>
```

| Field | Values | Example | Meaning |
|---|---|---|---|
| Minute | 0-59 | `30` | Minute 30 |
| Hour | 0-23 | `2` | 02:00 |
| Day of month | 1-31 | `15` | 15th day |
| Month | 1-12 | `*` | Every month |
| Day of week | 0-7 (0 and 7 are Sunday) | `1` | Monday |

Example entries:

```text
0 2 * * * /usr/local/bin/backup.sh
*/15 * * * * /usr/local/bin/check.sh
30 2 * * 1 /usr/local/bin/weekly.sh
0 0 1 * * /usr/local/bin/monthly.sh
```

**Field modifiers:**
- `*`: Matches every value.
- `,`: Separates values in a list (`1,15`).
- `-`: Defines a range (`1-5`).
- `/`: Defines a step (`*/15`).

Cron runs with a minimal environment. Set `PATH` at the top of the crontab when the command depends on it.

## 4. Use System-Wide Cron Files

`/etc/crontab` and files in `/etc/cron.d/` add a user field before the command:

```text
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
0 2 * * * root /usr/local/bin/backup.sh
```

Create a custom schedule with the same format in `/etc/cron.d/<JOB_NAME>`. The directories `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` execute contained scripts at the named intervals. User crontabs are stored in `/var/spool/cron/crontabs/`. Do not edit those files directly; use `crontab`.

## 5. Restrict Cron Access

Create the access files as root and list one user per line:

```bash
vim /etc/cron.allow
vim /etc/cron.deny
```

**Options:**
- `/etc/cron.allow`: If present, only the listed users may use `crontab`, one user per line.
- `/etc/cron.deny`: Blocks the listed users. Applies only when `cron.allow` is absent.
- If neither file exists, only root may use `crontab`.

These files restrict the `crontab` command only. They do not stop the execution of existing crontabs.

## 6. Check Cron Logs

Debian-based systems log cron through syslog and the systemd journal. Query the journal:

```bash
journalctl -t cron
journalctl -t cron --since "1 hour ago"
```

The same entries are written to `/var/log/syslog` and, depending on the rsyslog configuration, to `/var/log/auth.log`:

```bash
grep CRON /var/log/syslog
```

Expected output:

```text
Feb 10 02:00:01 host CRON[1234]: (root) CMD (/usr/local/bin/backup.sh)
```

## 7. Schedule with Systemd Timers

Create a service unit `/etc/systemd/system/<SERVICE_NAME>.service`:

```ini
[Unit]
Description=Nightly backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
```

Create the matching timer `/etc/systemd/system/<SERVICE_NAME>.timer`:

```ini
[Unit]
Description=Run the backup job nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start the timer:

```bash
systemctl daemon-reload
systemctl enable --now <SERVICE_NAME>.timer
```

**Options:**
- `Type=oneshot`: Runs the command once and exits.
- `OnCalendar=`: Calendar event expression. Format `DayOfWeek Year-Month-Day Hour:Minute:Second`.
- `Persistent=true`: Runs the job immediately after boot if a scheduled run was missed.
- `WantedBy=timers.target`: Activates the timer together with the other system timers.

## 8. Verify the Schedule

List active timers, test a calendar expression, and inspect the triggered service logs:

```bash
systemctl list-timers
systemd-analyze calendar "*-*-* 02:00:00"
journalctl -u <SERVICE_NAME>.service
```

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install Cron

Run as root:

```bash
dnf install cronie
systemctl enable --now crond
```

The service is named `crond`. Verify:

```bash
systemctl status crond
```

## 2. Edit the User Crontab

Open the crontab editor, list entries, or remove the crontab:

```bash
crontab -e
crontab -l
crontab -r
```

**Options:**
- `-e`: Opens the crontab in the editor set by `EDITOR`.
- `-l`: Lists the current user's crontab.
- `-r`: Removes the current user's crontab without a confirmation prompt. Use `crontab -ri` to confirm first.
- `-u <USERNAME>`: Operates on another user's crontab. Requires root.

## 3. Schedule a Job

The five-field syntax is identical to Debian-based systems. Example entries:

```text
0 2 * * * /usr/local/bin/backup.sh
*/15 * * * * /usr/local/bin/check.sh
30 2 * * 1 /usr/local/bin/weekly.sh
```

Field modifiers (`*` every value, `,` lists, `-` ranges, `/` steps) are identical to Debian-based systems. Set `PATH` explicitly at the top of the crontab and use absolute paths in commands.

## 4. Use System-Wide Cron Files

`/etc/crontab` and files in `/etc/cron.d/` add a user field before the command:

```text
0 2 * * * root /usr/local/bin/backup.sh
```

The directories `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, and `/etc/cron.monthly/` execute contained scripts at the named intervals. User crontabs are stored in `/var/spool/cron/`. Do not edit those files directly; use `crontab`.

## 5. Restrict Cron Access

Create the access files as root and list one user per line:

```bash
vim /etc/cron.allow
vim /etc/cron.deny
```

**Options:**
- `/etc/cron.allow`: If present, only the listed users may use `crontab`.
- `/etc/cron.deny`: Blocks the listed users. Applies only when `cron.allow` is absent.
- RHEL-based systems commonly ship `/etc/cron.deny`. If neither file exists, only root may use `crontab`.

These files restrict the `crontab` command only. They do not stop the execution of existing crontabs.

## 6. Check Cron Logs

Cronie writes to `/var/log/cron` and to the journal with the syslog identifier `crond`:

```bash
journalctl -t crond --since "1 hour ago"
grep CROND /var/log/cron
```

Expected output:

```text
Feb 10 02:00:01 host CROND[1234]: (root) CMD (/usr/local/bin/backup.sh)
```

## 7. Schedule with Systemd Timers

Timer units work exactly as on Debian-based systems. Create the service and timer pair:

```ini
# /etc/systemd/system/<SERVICE_NAME>.service
[Unit]
Description=Nightly backup job

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

# /etc/systemd/system/<SERVICE_NAME>.timer
[Unit]
Description=Run the backup job nightly

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable and start the timer:

```bash
systemctl daemon-reload
systemctl enable --now <SERVICE_NAME>.timer
```

**Options:**
- `Type=oneshot`: Runs the command once and exits.
- `OnCalendar=`: Calendar event expression. Format `DayOfWeek Year-Month-Day Hour:Minute:Second`.
- `Persistent=true`: Runs the job immediately after boot if a scheduled run was missed.
- `WantedBy=timers.target`: Activates the timer together with the other system timers.

## 8. Verify the Schedule

```bash
systemctl list-timers
systemd-analyze calendar "*-*-* 02:00:00"
journalctl -u <SERVICE_NAME>.service
```

---

# Part 3: Important Notes

## Cron vs. Systemd Timers

| Property | Cron | Systemd timers |
|---|---|---|
| Configuration | Crontab lines | `.timer` and `.service` unit files |
| Dependency ordering | Not supported | `After=`, `Requires=` |
| Missed runs | Skipped by default | `Persistent=true` runs missed jobs |
| Logging | Syslog and journal | Journald through the service unit |
| Randomized delay | Not built in | `RandomizedDelaySec=` |
| Verification | `crontab -l` | `systemctl list-timers` |
| Access control | `cron.allow` and `cron.deny` | Systemd unit permissions |

## Cron Logging Comparison

| Location | Debian-based | RHEL-based |
|---|---|---|
| Journal identifier | `cron` | `crond` |
| Log file | `/var/log/syslog`, `/var/log/auth.log` | `/var/log/cron` |
| Journal query | `journalctl -t cron` | `journalctl -t crond` |

## Scheduling Guidelines

- Use absolute paths for commands and scripts in every job. Cron does not inherit the interactive shell environment.
- Set `PATH` explicitly at the top of the crontab when the command depends on non-standard locations.
- Redirect job output to a log file, for example `> /var/log/<JOB_NAME>.log 2>&1`, so failures are visible.
- Do not schedule the same job in a crontab and a systemd timer. The job will run twice.
- Use `Persistent=true` for timers whose jobs must run after downtime, and `RandomizedDelaySec=` to avoid synchronized load.
- Run jobs as a dedicated user with `crontab -u <USERNAME> -e` or `User=` in the service unit.
- Keep jobs idempotent. A repeated run must not corrupt state.

## Security Guidelines

- Replace placeholders such as `<USERNAME>`, `<SERVICE_NAME>`, and `<JOB_NAME>` with real names. Do not include the angle brackets in commands.
- Restrict cron with `/etc/cron.allow` on multi-user systems. Deny-by-default beats a long `cron.deny`.
- Run scheduled jobs as dedicated service accounts instead of root whenever possible.
- Protect scripts invoked by cron with restrictive permissions. Crontab entries and script bodies can expose credentials.
- Prefer `sudo` rules with exact command lists over root crontabs for privileged jobs.
- Monitor `/var/log/cron`, `/var/log/syslog`, and the journal for unexpected scheduled tasks.
- Audit with `crontab -l` and remove stale entries with `crontab -e`.
- Never edit files under `/var/spool/cron/` directly. Use `crontab`.
- Confirm activation times with `systemctl list-timers` after enabling any timer.
