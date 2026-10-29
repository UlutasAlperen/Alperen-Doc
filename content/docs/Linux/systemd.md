---
title: "systemd"
weight: 60
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** systemctl basics, unit file anatomy, custom services, journald logging, and startup analysis.

## 1. Check Service Status

Inspect a unit:

```bash
systemctl status <SERVICE_NAME>
```

Expected output:

```text
● <SERVICE_NAME>.service - Example application
     Loaded: loaded (/etc/systemd/system/<SERVICE_NAME>.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-02-10 08:00:00 UTC; 5min ago
   Main PID: 1234 (myapp)
      Tasks: 3
     Memory: 24.0M
```

Query individual states:

```bash
systemctl is-active <SERVICE_NAME>
systemctl is-enabled <SERVICE_NAME>
```

**Options:**
- `is-active`: Prints `active` or `inactive` for the current runtime state.
- `is-enabled`: Prints `enabled` or `disabled` for the boot-time state.

## 2. Start, Stop, and Restart Services

Run as root:

```bash
systemctl start <SERVICE_NAME>
systemctl stop <SERVICE_NAME>
systemctl restart <SERVICE_NAME>
systemctl reload <SERVICE_NAME>
```

**Options:**
- `start`: Starts the unit immediately.
- `stop`: Stops the unit and its remaining processes.
- `restart`: Stops and then starts the unit. Use after changing code or binaries.
- `reload`: Reloads the unit configuration without dropping open connections. Not all units support it.

## 3. Enable and Disable Services at Boot

Enable or disable a unit for future boots:

```bash
systemctl enable <SERVICE_NAME>
systemctl disable <SERVICE_NAME>
```

Combine enabling with immediate start:

```bash
systemctl enable --now <SERVICE_NAME>
```

**Options:**
- `--now`: Activates the unit immediately in addition to enabling it at boot.

Enabling a unit creates a symlink under `/etc/systemd/system/multi-user.target.wants/`. Verify:

```bash
systemctl is-enabled <SERVICE_NAME>
```

Expected output:

```text
enabled
```

## 4. Understand the Unit File Anatomy

Unit files use INI-style sections. The three mandatory sections are:

- `[Unit]`: Metadata and ordering. Describes the unit and its dependencies.
- `[Service]`: Process behavior. Defines how the service starts, runs, and stops.
- `[Install]`: Activation targets. Defines where the unit is enabled.

Complete example unit:

```ini
[Unit]
Description=Example application service
After=network.target

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/myapp --config /etc/myapp/myapp.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Options:**
- `After=network.target`: Orders the unit after the network stack is up. It does not pull the target in.
- `Type=simple`: The process started by `ExecStart` is the main service process.
- `User=` and `Group=`: Run the service under a dedicated account instead of root.
- `Restart=on-failure`: Restarts the unit when it exits with a non-zero status.
- `RestartSec=5`: Waits five seconds before restarting.
- `WantedBy=multi-user.target`: Enables the unit for a normal multi-user boot.

## 5. Install a Custom Unit

Create the unit file:

```bash
vim /etc/systemd/system/<SERVICE_NAME>.service
```

Set ownership and permissions:

```bash
chown root:root /etc/systemd/system/<SERVICE_NAME>.service
chmod 644 /etc/systemd/system/<SERVICE_NAME>.service
```

Place custom units in `/etc/systemd/system/`. Units shipped by packages live in `/lib/systemd/system/` on Debian-based systems and `/usr/lib/systemd/system/` on RHEL-based systems. Do not edit package-provided files: updates overwrite them.

## 6. Reload the Manager

Run as root after every unit change:

```bash
systemctl daemon-reload
```

systemd does not detect modified unit files automatically. Then enable and start the service:

```bash
systemctl enable --now <SERVICE_NAME>
```

## 7. Verify the Service

Check the states:

```bash
systemctl is-active <SERVICE_NAME>
systemctl is-enabled <SERVICE_NAME>
```

Confirm the main process and its state:

```bash
systemctl show <SERVICE_NAME> -p MainPID -p ActiveState -p SubState
```

Expected output:

```text
MainPID=1234
ActiveState=active
SubState=running
```

## 8. Read Service Logs with Journald

Show all logs for one unit:

```bash
journalctl -u <SERVICE_NAME>
```

Jump to the most recent entries:

```bash
journalctl -u <SERVICE_NAME> -e
```

Filter by time:

```bash
journalctl -u <SERVICE_NAME> --since "2026-02-10 08:00:00"
journalctl -u <SERVICE_NAME> --since "1 hour ago" --until "now"
```

Follow new entries in real time:

```bash
journalctl -u <SERVICE_NAME> -f
```

Check how much disk space the journal occupies:

```bash
journalctl --disk-usage
```

**Options:**
- `-u`: Filters entries by unit name.
- `-e`: Jumps to the end of the journal, showing only the newest entries.
- `--since` and `--until`: Restrict entries to a time range. Accept timestamps and relative expressions.
- `-f`: Follows new entries, similar to `tail -f`.
- `--disk-usage`: Prints the disk space used by journal files.

## 9. Analyze Startup Time

Measure boot performance:

```bash
systemd-analyze
systemd-analyze blame
```

Inspect the critical path:

```bash
systemd-analyze critical-chain
```

`systemd-analyze blame` lists units sorted by initialization time. Use it to identify slow services that delay the boot.

---

# Part 3: Important Notes

## systemctl Command Reference

| Operation | Command | Effect |
|---|---|---|
| Check status | `systemctl status <SERVICE_NAME>` | Shows state, PID, and recent logs |
| Start | `systemctl start <SERVICE_NAME>` | Starts the unit now |
| Stop | `systemctl stop <SERVICE_NAME>` | Stops the unit now |
| Restart | `systemctl restart <SERVICE_NAME>` | Stops and starts the unit |
| Reload configuration | `systemctl reload <SERVICE_NAME>` | Reloads without a full restart |
| Enable at boot | `systemctl enable <SERVICE_NAME>` | Adds the unit to boot targets |
| Disable at boot | `systemctl disable <SERVICE_NAME>` | Removes the unit from boot targets |
| Reload manager | `systemctl daemon-reload` | Reloads modified unit files |

## Journal Queries

| Goal | Command |
|---|---|
| Logs of one unit | `journalctl -u <SERVICE_NAME>` |
| Newest entries | `journalctl -u <SERVICE_NAME> -e` |
| Entries since a point in time | `journalctl -u <SERVICE_NAME> --since "1 hour ago"` |
| Live log stream | `journalctl -u <SERVICE_NAME> -f` |
| Journal storage usage | `journalctl --disk-usage` |

## Unit File Locations

- `/etc/systemd/system/`: Administrator-created units and overrides. Highest precedence.
- `/lib/systemd/system/`: Vendor units on Debian-based systems.
- `/usr/lib/systemd/system/`: Vendor units on RHEL-based systems.
- `/run/systemd/system/`: Runtime units. Cleared at every boot.

## Security Guidelines

- Replace placeholders such as `<SERVICE_NAME>` with actual unit names. Do not include the angle brackets in commands.
- Run `systemctl` management commands as root or through `sudo`.
- Keep custom units in `/etc/systemd/system/`. Never edit files under `/lib/systemd/system/` or `/usr/lib/systemd/system/`; package updates replace them.
- Run services as a dedicated non-root account using `User=` and `Group=`.
- Restrict unit file permissions to `644` with root ownership. Unit files can execute arbitrary commands as root.
- Combine `Restart=` with `StartLimitIntervalSec=` and `StartLimitBurst=` to avoid restart loops on broken services.
- Run `systemctl daemon-reload` after every unit change and verify the result with `systemctl status`.
- Limit journal growth with `SystemMaxUse=` in `/etc/systemd/journald.conf`; logs may contain sensitive data and can fill the root filesystem.
- Restrict journal reading to administrators. Read access to the journal exposes service output for the whole host.
