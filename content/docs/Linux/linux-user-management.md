---
title: "linux-user-management"
weight: 10
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Regular user creation, system user creation, sudo privileges, verification, and usage scenarios.

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install sudo

Run as root:

```bash
apt update
apt install sudo
```

## 2. Create a Regular User

Create a user interactively:

```bash
adduser <USERNAME>
```

Alternatively, use `useradd`:

```bash
useradd -m -s /bin/bash <USERNAME>
passwd <USERNAME>
```

**Options:**
- `-m`: Creates the user's home directory.
- `-s /bin/bash`: Sets Bash as the default shell.
- `passwd`: Sets or changes the user's password.

## 3. Grant Sudo Privileges

Add the user to the `sudo` group:

```bash
usermod -aG sudo <USERNAME>
```

Verify group membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Confirm that `sudo` appears in the output.

Validate the sudoers configuration:

```bash
visudo -c
```

The default configuration should authorize the `sudo` group. If necessary, inspect it using:

```bash
visudo
```

The relevant rule is typically:

```text
%sudo ALL=(ALL:ALL) ALL
```

Do not enable this rule without understanding its implications: it grants group members broad administrative privileges.

## 4. Create a System User

System accounts are intended for services and applications rather than interactive human logins.

Create a system account with its own group, no home directory, and a non-login shell:

```bash
adduser --system --group --no-create-home --shell /usr/sbin/nologin <SERVICE_USER>
```

Verify the account:

```bash
id <SERVICE_USER>
getent passwd <SERVICE_USER>
command -v nologin
```

Check that the account has the expected shell and that it cannot normally open an interactive login shell.

### System user usage scenarios

**Scenario A: Application service**

```bash
adduser --system --group --no-create-home --shell /usr/sbin/nologin myapp
```

A systemd service can run under this account:

```ini
[Service]
User=myapp
Group=myapp
ExecStart=/opt/myapp/bin/myapp
```

The executable and required files must be accessible to `myapp`.

**Scenario B: Backup service**

```bash
adduser --system --group --no-create-home --shell /usr/sbin/nologin backupsvc
```

Grant `backupsvc` access only to the files and directories required for its backup operations.

**Scenario C: Monitoring exporter**

```bash
adduser --system --group --no-create-home --shell /usr/sbin/nologin exporter
```

Grant the account only the read permissions needed to collect monitoring data.

## 5. Test Sudo Privileges

Switch to the regular user:

```bash
su - <USERNAME>
```

Inspect the permitted operations:

```bash
sudo -l
```

Test administrative execution:

```bash
sudo whoami
```

Expected output:

```text
root
```

If group membership was changed after login, start a new login session before testing.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install sudo

Run as root:

```bash
dnf install sudo
```

If sudo is already installed, this step is unnecessary.

## 2. Create a Regular User

Create the user and home directory:

```bash
useradd -m -s /bin/bash <USERNAME>
```

Set the password:

```bash
passwd <USERNAME>
```

**Options:**
- `-m`: Creates the user's home directory.
- `-s /bin/bash`: Sets Bash as the default shell.

## 3. Grant Sudo Privileges

RHEL-based systems commonly use the `wheel` group for administrative access.

Add the user to the group:

```bash
usermod -aG wheel <USERNAME>
```

Verify group membership:

```bash
id <USERNAME>
groups <USERNAME>
```

Validate the sudoers configuration:

```bash
visudo -c
```

If necessary, inspect or edit the configuration using:

```bash
visudo
```

The following rule is commonly used to authorize the `wheel` group:

```text
%wheel ALL=(ALL) ALL
```

Verify that the rule is enabled and is not overridden by another applicable configuration rule.

## 4. Create a System User

Create a system account with its own group, no home directory, and a non-login shell:

```bash
useradd --system --user-group --no-create-home --shell /sbin/nologin <SERVICE_USER>
```

Verify the account:

```bash
id <SERVICE_USER>
getent passwd <SERVICE_USER>
command -v nologin
```

Check that the account has the expected shell and that it cannot normally open an interactive login shell.

### System user usage scenarios

**Scenario A: Application service**

```bash
useradd --system --user-group --no-create-home --shell /sbin/nologin myapp
```

Example systemd service configuration:

```ini
[Service]
User=myapp
Group=myapp
ExecStart=/opt/myapp/bin/myapp
```

**Scenario B: Backup service**

```bash
useradd --system --user-group --no-create-home --shell /sbin/nologin backupsvc
```

Grant only the permissions required by the backup process.

**Scenario C: Monitoring exporter**

```bash
useradd --system --user-group --no-create-home --shell /sbin/nologin exporter
```

Grant read access only to the resources the exporter needs to monitor.

## 5. Test Sudo Privileges

Switch to the regular user:

```bash
su - <USERNAME>
```

Check sudo permissions:

```bash
sudo -l
```

Test administrative execution:

```bash
sudo whoami
```

Expected output:

```text
root
```

If group membership was changed after login, start a new login session before testing.

---

# Part 3: Important Notes

## Regular Users vs. System Users

| Property | Regular user | System user |
|---|---|---|
| Intended purpose | Human login and administrative work | Application or service execution |
| Home directory | Usually present | Often omitted |
| Interactive shell | Usually enabled | Commonly disabled |
| Sudo privileges | Only when explicitly granted | Not granted by default |
| Example | `<USERNAME>` | `myapp`, `backupsvc`, `exporter` |

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Install sudo | `apt install sudo` | `dnf install sudo` |
| Create regular user | `adduser <USERNAME>` | `useradd -m -s /bin/bash <USERNAME>` |
| Set password | `passwd <USERNAME>` | `passwd <USERNAME>` |
| Grant sudo access | `usermod -aG sudo <USERNAME>` | `usermod -aG wheel <USERNAME>` |
| Create system user | `adduser --system --group --no-create-home --shell /usr/sbin/nologin <SERVICE_USER>` | `useradd --system --user-group --no-create-home --shell /sbin/nologin <SERVICE_USER>` |
| Check account | `id <USERNAME>` | `id <USERNAME>` |
| Check sudo configuration | `visudo -c` | `visudo -c` |
| Test sudo | `sudo whoami` | `sudo whoami` |

## Security Guidelines

- Replace placeholders such as `<USERNAME>` and `<SERVICE_USER>` with actual account names. Do not include the angle brackets in commands.
- Always use `usermod -aG` when adding supplementary group membership to avoid replacing existing supplementary groups.
- System accounts should not be added to `sudo` or `wheel unless the service has a specific, justified requirement for administrative privileges.
- Use a dedicated account for each application or service where practical.
- Grant only the file permissions and capabilities required for the account's purpose.
- Use `visudo` to edit sudoers configuration safely.
- When logged in as root, the user-management commands above generally do not require the `sudo` prefix.
- Use a regular account for everyday administration and elevate privileges with `sudo` when necessary.
