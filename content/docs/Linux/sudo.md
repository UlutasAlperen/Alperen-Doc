---
title: "sudo"
weight: 40
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** sudo basics, visudo, sudoers syntax, command tags, drop-in files, privilege inspection, logging, and common pitfalls.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install sudo

Run as root:

```bash
apt update
apt install sudo
```

## 2. Edit the Sudoers File Safely

Edit the sudoers configuration only with `visudo`:

```bash
visudo
```

`visudo` locks the file against concurrent edits, parses the result before saving, and refuses to write a broken configuration. A single syntax error in `/etc/sudoers` can remove administrative access from every account.

The default configuration authorizes the `sudo` group:

```text
%sudo ALL=(ALL:ALL) ALL
```

Read each rule as `who where = (as whom) what`:

- `%sudo`: All members of the `sudo` group.
- `ALL`: On every host.
- `(ALL:ALL)`: As any user and any group.
- `ALL`: Running any command.

Do not broaden this rule. Narrow it whenever a specific task justifies a narrower grant.

## 3. Grant Access to the sudo Group

Add the user to the `sudo` group:

```bash
usermod -aG sudo <USERNAME>
```

Verify the membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Confirm that `sudo` appears in the output, then start a new login session before testing.

## 4. Create a Drop-In File

Keep task-specific rules out of the main file. Create a drop-in file with `visudo` so the syntax is checked:

```bash
visudo -f /etc/sudoers.d/10-deploy
```

Example content:

```text
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

Expected result: the `deploy` account can restart one service without a password and nothing else.

## 5. Validate the Configuration

Check the main file and all drop-in files:

```bash
visudo -c
```

Expected output:

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/10-deploy: parsed OK
```

Run this check after every change. Any line that does not report `parsed OK` must be fixed immediately.

## 6. Inspect Privileges

Switch to the user and inspect the effective permissions:

```bash
su - <USERNAME>
sudo -l
```

Expected output:

```text
User alice may run the following commands on debian-host:
    (root) NOPASSWD: /usr/bin/systemctl restart myapp
```

Test the permitted command:

```bash
sudo /usr/bin/systemctl restart myapp
```

Test denied commands as well. A rule that grants more than intended is a defect.

## 7. Review sudo Logs

Authentication and authorization events are logged to `/var/log/auth.log`. Search for sudo activity:

```bash
grep sudo /var/log/auth.log
```

Query the systemd journal instead when the text log is unavailable:

```bash
journalctl -t sudo
```

Filter by user or time window:

```bash
journalctl -t sudo --since today
```

Verify that every use of `sudo` produces a log entry. Silent execution indicates a logging problem.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install sudo

Run as root:

```bash
dnf install sudo
```

If sudo is already installed, this step is unnecessary.

## 2. Edit the Sudoers File Safely

Edit the sudoers configuration only with `visudo`:

```bash
visudo
```

`visudo` locks the file, checks the syntax before saving, and refuses to write a broken configuration. RHEL-based systems commonly authorize the `wheel` group:

```text
%wheel ALL=(ALL) ALL
```

Read the rule as `who where = (as whom) what`:

- `%wheel`: All members of the `wheel` group.
- `ALL`: On every host.
- `(ALL)`: As any user.
- `ALL`: Running any command.

Verify that the rule is active and is not commented out or overridden by a later rule.

## 3. Grant Access to the wheel Group

Add the user to the `wheel` group:

```bash
usermod -aG wheel <USERNAME>
```

Verify the membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Confirm that `wheel` appears in the output, then start a new login session before testing.

## 4. Create a Drop-In File

Keep task-specific rules out of the main file. Create a drop-in file with `visudo`:

```bash
visudo -f /etc/sudoers.d/10-deploy
```

Example content:

```text
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

Expected result: the `deploy` account can restart one service without a password and nothing else.

## 5. Validate the Configuration

Check the main file and all drop-in files:

```bash
visudo -c
```

Expected output:

```text
/etc/sudoers: parsed OK
/etc/sudoers.d/10-deploy: parsed OK
```

Run this check after every change.

## 6. Inspect Privileges

Switch to the user and inspect the effective permissions:

```bash
su - <USERNAME>
sudo -l
```

Expected output:

```text
User deploy may run the following commands on rhel-host:
    (root) NOPASSWD: /usr/bin/systemctl restart myapp
```

Test the permitted command and confirm that unintended commands are denied.

## 7. Review sudo Logs

sudo events are recorded in the systemd journal. Search for sudo activity:

```bash
journalctl -t sudo
```

Filter by user or time window:

```bash
journalctl -t sudo --since today
```

To verify that the audit trail is complete, check for the audit daemon:

```bash
systemctl status auditd
```

Verify that every use of `sudo` produces a log entry.

---

# Part 3: Important Notes

## Sudoers Syntax Reference

One rule per line:

```text
who where = (runas_user:runas_group) commands
```

| Field | Meaning | Examples |
|---|---|---|
| `who` | User or group receiving the grant | `alice`, `%sudo`, `%wheel`, `%deploy` |
| `where` | Hosts where the rule applies | `ALL`, `web01` |
| `runas_user` | Target user for execution | `ALL`, `root` |
| `runas_group` | Target group for execution | `ALL`, `root` |
| `commands` | Allowed commands with absolute paths | `ALL`, `/usr/bin/systemctl` |

A `%` prefix makes the first field a group name. Usernames and group names are case-sensitive. Commands must use absolute paths because `sudo` does not resolve them through the invoking user's `PATH`.

## Command Tags

| Tag | Effect |
|---|---|
| `NOPASSWD:` | No password is required for the listed commands |
| `PASSWD:` | Password is required; this is the default |
| `NOEXEC:` | Prevents the command from executing further programs |
| `SETENV:` | Allows the caller to set environment variables |
| `!` prefix | Negates a command or tag |

Tags apply to the commands that follow them on the same rule. Place `NOPASSWD:` directly in front of the smallest possible command list.

## Drop-In File Rules

- Drop-in files live in `/etc/sudoers.d/` and are read through `@includedir /etc/sudoers.d` in the main file. Older configurations use `#includedir`.
- Files are processed in lexicographic order. Later rules override earlier ones for the same user and host.
- Name files with a numeric prefix and no dots, for example `10-deploy` or `90-backup`. Files containing a dot or ending in `~` are ignored.
- Edit drop-in files with `visudo -f /etc/sudoers.d/<FILE_NAME>` so the syntax is validated.
- Remove obsolete files. An abandoned rule is a permanent privilege grant.

## Common Pitfalls

- Editing `/etc/sudoers` with `vim` or a configuration-management push without `visudo`. A broken file leaves the host without administrative access and requires console recovery.
- Granting `ALL=(ALL) NOPASSWD: ALL`. This gives permanent passwordless root and provides no audit value beyond a single log line.
- Writing commands without absolute paths, for example `systemctl restart myapp`. The rule silently fails to match.
- Forgetting that group membership changes require a new login session before `sudo -l` reflects them.
- Leaving default editor sessions open while `visudo` holds its lock, which blocks other administrators.
- Relying on `sudo` logging without verifying that `rsyslog` or `journald` is running and that log storage is persistent.

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Install sudo | `apt install sudo` | `dnf install sudo` |
| Edit sudoers safely | `visudo` | `visudo` |
| Edit a drop-in file | `visudo -f /etc/sudoers.d/10-deploy` | `visudo -f /etc/sudoers.d/10-deploy` |
| Validate configuration | `visudo -c` | `visudo -c` |
| Grant admin access | `usermod -aG sudo <USERNAME>` | `usermod -aG wheel <USERNAME>` |
| Inspect privileges | `sudo -l` | `sudo -l` |
| Test execution | `sudo whoami` | `sudo whoami` |
| Review logs | `grep sudo /var/log/auth.log` | `journalctl -t sudo` |
| Review journal | `journalctl -t sudo` | `journalctl -t sudo` |

## Security Guidelines

- Replace placeholders such as `<USERNAME>` and `<FILE_NAME>` with actual values. Do not include the angle brackets in commands.
- Always edit sudoers content with `visudo` or `visudo -f <FILE>`. Never edit the files with a plain editor or push unvalidated content to them.
- Validate every change with `visudo -c` before logging out of the session.
- Grant the narrowest rule that covers the task. Prefer a single command over a command directory and a directory over `ALL`.
- Use `NOPASSWD` only for automation accounts that cannot type a password, and restrict those accounts to the exact commands they need.
- Do not grant `%sudo` or `%wheel` membership to service or system accounts.
- Prefer drop-in files in `/etc/sudoers.d/` over modifying the main file. Name them with a numeric prefix and no dots.
- Remove rules and drop-in files when the task or role that justified them ends.
- Review `sudo -l` output for each privileged account during access audits.
- Use a regular account for daily work and elevate with `sudo` only when required. Avoid working in a root shell.
