---
title: "selinux-apparmor"
weight: 180
---
**Scope:** Mandatory access control on RHEL-based and Debian-based Linux distributions  
**Topics:** SELinux modes, file contexts, booleans, denial analysis, AppArmor profiles, troubleshooting.

Traditional Unix permissions implement discretionary access control (DAC): the file owner decides who may read, write, or execute a resource. Mandatory access control (MAC) adds a system-wide policy that restricts programs to only the resources they are explicitly allowed to use, regardless of file ownership. SELinux enforces MAC with labels on processes and objects, while AppArmor enforces it with path-based profiles.

# Part 1: SELinux (RHEL-Based Systems)

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Check the SELinux Mode

Inspect the current mode:

```bash
getenforce
```

Expected output:

```text
Enforcing
```

Inspect the full status:

```bash
sestatus
```

Expected output:

```text
SELinux status:                 enabled
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
```

## 2. Change the Mode at Runtime

Run as root:

```bash
setenforce 0
setenforce 1
```

**Options:**
- `0`: Switches to permissive mode. Violations are logged but not blocked.
- `1`: Switches to enforcing mode. Violations are logged and blocked.

The change is runtime only and is lost on reboot. Use permissive mode to test whether SELinux is responsible for a failure.

## 3. Set the Mode Persistently

Inspect the configuration:

```bash
cat /etc/selinux/config
```

Example:

```text
SELINUX=enforcing
SELINUXTYPE=targeted
```

Edit `SELINUX=` to one of `enforcing`, `permissive`, or `disabled`. Changing `disabled` to an active mode requires a full filesystem relabel and a reboot:

```bash
touch /.autorelabel
reboot
```

## 4. Inspect File Contexts

Every file and process carries a security context:

```bash
ls -Z /var/www
```

Expected output:

```text
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```

Inspect process contexts with `ps -eZ | grep httpd`.

## 5. Restore File Contexts

Fix a wrong label temporarily:

```bash
chcon -t httpd_sys_content_t /srv/www/index.html
```

`chcon` changes are lost when contexts are relabeled. For a persistent mapping, define a file context rule and apply it:

```bash
semanage fcontext -a -t httpd_sys_content_t "/srv/www(/.*)?"
restorecon -Rv /srv/www
```

Expected output:

```text
Relabeled /srv/www/index.html (unconfined_u:object_r:default_t:s0 -> unconfined_u:object_r:httpd_sys_content_t:s0)
```

Verify the mapping with `semanage fcontext -l | grep /srv/www` and `ls -Z /srv/www`.

**Options:**
- `semanage fcontext -a`: Adds a persistent context mapping.
- `-t <TYPE>`: Sets the target type, for example `httpd_sys_content_t`.
- `"/srv/www(/.*)?"`: Matches the directory and everything below it.
- `restorecon -Rv`: Restores contexts recursively and prints each change.

On RHEL 9, `semanage` is provided by `policycoreutils-python-utils`:

```bash
dnf install policycoreutils-python-utils
```

## 6. Manage Booleans

List booleans related to a service or inspect a single one:

```bash
getsebool -a | grep httpd
getsebool httpd_can_network_connect
```

Enable a boolean persistently:

```bash
setsebool -P httpd_can_network_connect on
```

**Options:**
- `-P`: Writes the change to policy so it survives a reboot.
- Without `-P`: Applies the change only until the next reboot.

## 7. Diagnose Denials

Search the audit log for access vector cache denials:

```bash
ausearch -m avc -ts recent
```

Explain the denials:

```bash
ausearch -m avc -ts today | audit2why
```

Expected output:

```text
type=AVC msg=audit(...): avc:  denied  { read } for  pid=1234 comm="httpd" ...
    was caused by:
        Missing type enforcement (TE) allow rule.
```

For detailed suggestions, install `setroubleshoot-server` and run `sealert -a /var/log/audit/audit.log`.

`ausearch` requires the audit daemon:

```bash
systemctl enable --now auditd
```

---

# Part 2: AppArmor (Debian-Based Systems)

**Examples:** Debian, Ubuntu

## 1. Check AppArmor Status

Run as root:

```bash
aa-status
```

Expected output:

```text
apparmor module is loaded.
18 profiles are in enforce mode.
3 profiles are in complain mode.
```

Install the tools if the command is missing:

```bash
apt install apparmor apparmor-utils
```

## 2. Inspect Profiles

List the loaded profiles:

```bash
ls /etc/apparmor.d
```

Each profile file is named after the executable path with `/` replaced by `.`. Inspect one:

```bash
cat /etc/apparmor.d/usr.sbin.tcpdump
```

## 3. Switch Between Enforce and Complain Mode

Put a profile into complain mode for testing:

```bash
aa-complain /etc/apparmor.d/usr.sbin.mysqld
```

Return it to enforce mode:

```bash
aa-enforce /etc/apparmor.d/usr.sbin.mysqld
```

Disable a profile completely:

```bash
aa-disable /etc/apparmor.d/usr.sbin.mysqld
```

**Options:**
- Enforce mode: Violations are logged and blocked.
- Complain mode: Violations are logged but not blocked. Use it to gather a profile before enforcing.
- `aa-disable`: Unloads the profile and stops it from loading at boot.

## 4. Create a Minimal Profile

Create a profile for a custom service:

```bash
touch /etc/apparmor.d/usr.sbin.myappd
```

Example content:

```text
#include <tunables/global>

/usr/sbin/myappd {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  capability net_bind_service,

  /etc/myapp.conf r,
  /var/lib/myapp/** rwk,
  /var/log/myapp.log w,
}
```

Load the profile:

```bash
apparmor_parser -r /etc/apparmor.d/usr.sbin.myappd
```

Alternatively, generate a profile interactively with `aa-genprof /usr/sbin/myappd` and exercise the application while it runs so that the required accesses are recorded.

## 5. Inspect Denial Logs

Search the kernel log for AppArmor events:

```bash
journalctl -k | grep apparmor
```

Expected output:

```text
audit: type=1400 audit(...): apparmor="DENIED" operation="open" profile="/usr/sbin/myappd" name="/srv/data/file" comm="myappd"
```

Filter only denials:

```bash
journalctl | grep 'apparmor="DENIED"'
```

---

# Part 3: Important Notes

## Permission Denied with Correct Unix Permissions

When `ls -l` shows correct ownership and mode but access still fails, follow this diagnostic flow:

- Confirm the DAC layer first: `ls -l <PATH>` and `namei -l <PATH>` for the full path chain.
- Identify the MAC layer in use: `getenforce` on RHEL-based systems, `aa-status` on Debian-based systems.
- Reproduce the failure and capture the denial: `ausearch -m avc -ts recent` for SELinux, `journalctl -k | grep apparmor` for AppArmor.
- Confirm the MAC layer is responsible: switch to permissive (`setenforce 0`) or complain (`aa-complain <PROFILE>`), reproduce again, and compare.
- Fix the policy instead of the symptom: correct file contexts with `restorecon`, add a persistent mapping with `semanage fcontext`, enable a boolean with `setsebool -P`, or extend the AppArmor profile.
- Restore enforcing or enforce mode and verify that the application still works.

## SELinux vs. AppArmor

| Property | SELinux | AppArmor |
|---|---|---|
| Common defaults | RHEL, Rocky Linux, AlmaLinux, Fedora | Debian, Ubuntu, SUSE |
| Enforcement model | Labels on processes and files | Path-based profiles |
| Main configuration | Policy modules, file contexts, booleans | Text profiles in `/etc/apparmor.d` |
| Switch to learning mode | `setenforce 0` (system-wide) | `aa-complain <PROFILE>` (per profile) |
| Inspect denials | `ausearch -m avc`, `audit2why`, `sealert` | `journalctl -k`, `dmesg` |
| Persistent disable | `SELINUX=disabled` in `/etc/selinux/config` and reboot | `aa-disable <PROFILE>` |
| Profile generation | `audit2allow` from denials | `aa-genprof`, `aa-logprof` |

## Command Reference

| Operation | SELinux | AppArmor |
|---|---|---|
| Check status | `getenforce`, `sestatus` | `aa-status` |
| Switch to learning mode | `setenforce 0` | `aa-complain <PROFILE>` |
| Enforce policy | `setenforce 1` | `aa-enforce <PROFILE>` |
| Inspect labels or profiles | `ls -Z`, `ps -eZ` | `ls /etc/apparmor.d` |
| Fix access rules | `restorecon`, `semanage fcontext` | Edit profile, `apparmor_parser -r` |
| Analyze denials | `ausearch -m avc -ts recent` | `journalctl -k \| grep apparmor` |

## Security Guidelines

- Do not disable SELinux or AppArmor to fix a denial. Fix the policy and keep MAC enabled.
- Use permissive or complain mode only for testing, and return to enforcing mode afterwards.
- Prefer `semanage fcontext` plus `restorecon` over `chcon`. `chcon` changes are lost on the next relabel.
- Use `setsebool -P` for permanent boolean changes. Without `-P` the change is lost on reboot.
- Keep profiles and policy changes minimal: allow only the paths and capabilities the application needs.
- Record the reason for every policy change and remove temporary exceptions after troubleshooting.
- Install `auditd` and keep audit logs long enough to reconstruct past denials.
- Regenerate or reload policy after every change and retest the affected application.
- Run `aa-status` and `sestatus` as part of routine server audits to verify the expected mode.