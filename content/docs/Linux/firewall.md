---
title: "firewall"
weight: 110
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Host firewalls, ufw, application profiles, firewalld, zones, rich rules, nftables, iptables, common ports, and safe rule management.

A host firewall filters incoming and outgoing traffic on a single system. It is the first control point for exposed services and should be configured before a service is started on a production interface. Debian-based systems commonly use ufw, RHEL-based systems use firewalld, and both build their rules on the nftables kernel framework.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Install ufw

Run as root:

```bash
apt update
apt install ufw
```

## 2. Inspect the Current Rules

```bash
sudo ufw status verbose
```

Expected output on a fresh system:

```text
Status: inactive
Default: deny (incoming), allow (outgoing), disabled (routed)
Logging: off
```

## 3. Set the Default Policies

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```

Deny incoming and allow outgoing is the standard baseline for servers.

## 4. Allow Traffic by Port

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.0.2.0/24 to any port 5432 proto tcp
```

**Options:**
- `allow`: Accepts matching traffic. Use `deny` to drop it.
- `22/tcp`: Port and protocol. Always specify the protocol for TCP and UDP services.
- `from <SOURCE>`: Restricts the rule to one source address or subnet.
- `to any port <PORT>`: Sets the destination port.
- `comment "<TEXT>"`: Attaches a note to the rule for later audits.

## 5. Allow an Application Profile

List the known profiles and inspect one:

```bash
sudo ufw app list
sudo ufw app info "OpenSSH"
```

Allow by profile name:

```bash
sudo ufw allow "OpenSSH"
```

## 6. Enable the Firewall

Allow SSH before enabling ufw:

```bash
sudo ufw allow 22/tcp
sudo ufw enable
```

Expected output:

```text
Firewall is active and enabled on system startup
```

Confirm the active ruleset:

```bash
sudo ufw status numbered
```

## 7. Delete Rules

List rules with numbers:

```bash
sudo ufw status numbered
```

Delete by number or by definition:

```bash
sudo ufw delete 3
sudo ufw delete allow 80/tcp
```

## 8. Enable Logging

```bash
sudo ufw logging on
sudo tail -f /var/log/ufw.log
```

Set the log level with `sudo ufw logging medium`. Logging records dropped packets and can be verbose on exposed hosts.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Install and Start firewalld

Run as root:

```bash
dnf install firewalld
systemctl enable --now firewalld
```

Verify the service:

```bash
systemctl status firewalld
```

## 2. Understand Zones

firewalld applies rules through zones. Each interface is bound to one zone, and the zone defines which traffic is accepted.

Show the default and active zones:

```bash
firewall-cmd --get-default-zone
firewall-cmd --get-active-zones
```

Expected output:

```text
public
  interfaces: eth0
```

## 3. Inspect the Rules

```bash
firewall-cmd --list-all
firewall-cmd --list-services
firewall-cmd --list-ports
```

## 4. Open Services and Ports

Add a rule for the current session only:

```bash
sudo firewall-cmd --add-service=ssh
sudo firewall-cmd --add-port=8080/tcp
```

Make the rule persistent:

```bash
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

Verify the result:

```bash
sudo firewall-cmd --list-all
```

**Options:**
- `--permanent`: Writes the rule to the configuration. Without it, the rule is lost at reload or reboot.
- `--reload`: Applies the permanent configuration to the runtime.
- `--add-service=<SERVICE>`: Opens the ports defined for a named service in `/usr/lib/firewalld/services/`.
- `--add-port=<PORT>/<PROTOCOL>`: Opens a single port.

## 5. Remove Rules

```bash
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --permanent --remove-port=8080/tcp
sudo firewall-cmd --reload
```

## 6. Add a Rich Rule

Restrict a database port to one source subnet:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.0.2.0/24" port protocol="tcp" port="5432" accept'
sudo firewall-cmd --reload
```

List rich rules:

```bash
sudo firewall-cmd --list-rich-rules
```

---

# Part 3: Important Notes

## nftables and iptables

Both ufw and firewalld generate nftables rules on current releases. `iptables` is a compatibility layer over the same kernel framework. Inspect the live ruleset with:

```bash
sudo nft list ruleset
sudo iptables -L -n -v
```

Install the tools if they are missing with `apt install nftables iptables` or `dnf install nftables iptables-nft`.

Do not mix manual `nft` or `iptables` rules with ufw or firewalld. The frontends overwrite or conflict with direct rules at every reload. Choose one management path per host.

## Common Ports

| Port | Protocol | Service | Typical use |
|---|---|---|---|
| 22 | TCP | SSH | Administration |
| 25 | TCP | SMTP | Mail relay |
| 53 | TCP/UDP | DNS | Name resolution |
| 123 | UDP | NTP | Time synchronization |
| 80 | TCP | HTTP | Web servers |
| 443 | TCP | HTTPS | Web servers |
| 3306 | TCP | MySQL/MariaDB | Database |
| 5432 | TCP | PostgreSQL | Database |
| 6443 | TCP | Kubernetes API server | Cluster control plane |
| 2379-2380 | TCP | etcd | Cluster control plane |
| 10250 | TCP | kubelet API | Cluster nodes |

## Command Reference

| Operation | ufw | firewalld |
|---|---|---|
| Show status | `ufw status verbose` | `firewall-cmd --list-all` |
| Open a port | `ufw allow 8080/tcp` | `firewall-cmd --permanent --add-port=8080/tcp` |
| Close a port | `ufw delete allow 8080/tcp` | `firewall-cmd --permanent --remove-port=8080/tcp` |
| Restrict a source | `ufw allow from <SOURCE> to any port 5432 proto tcp` | Rich rule with `source address` |
| Enable | `ufw enable` | `systemctl enable --now firewalld` |
| Reload | `ufw reload` | `firewall-cmd --reload` |
| Logging | `ufw logging on` | `firewall-cmd --set-log-denied=all` |

## Security Guidelines

- Open only the ports the host actually needs. Keep the default policy at deny incoming.
- Add allow rules before enabling the firewall. Enabling ufw without an SSH rule cuts off remote access.
- Persist every rule. ufw rules persist by default; firewalld rules require `--permanent` and `--reload`.
- Restrict management ports such as 22 and 6443 to trusted source addresses where possible.
- Avoid locking yourself out: keep console access, test changes in a second session before disconnecting, and prefer temporary rules during maintenance windows.
- Do not mix ufw, firewalld, and manual `nft` or `iptables` rules on one host.
- Review the active ruleset regularly with `ufw status numbered` or `firewall-cmd --list-all`.
- Keep database and cluster ports off public interfaces. Bound exposure to the networks that use them.
- Treat firewall logs as sensitive operational data and forward them to a central log system.
