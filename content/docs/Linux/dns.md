---
title: "dns"
weight: 100
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Resolution order, /etc/nsswitch.conf, /etc/hosts, /etc/resolv.conf, dig, nslookup, host, getent, systemd-resolved, NetworkManager DNS, caching, and search domains.

## 1. Inspect the Resolution Order

Inspect the Name Service Switch configuration:

```bash
cat /etc/nsswitch.conf
```

Locate the `hosts` line:

```text
hosts: files dns
```

`files` resolves names through `/etc/hosts`. `dns` queries the nameservers in `/etc/resolv.conf`. Sources are consulted from left to right.

## 2. Add Local Host Mappings

Edit the local hosts file:

```bash
sudo vim /etc/hosts
```

Format:

```text
127.0.0.1   localhost
192.0.2.10  db01.example.internal db01
192.0.2.20  app01.example.internal app01
```

Each line starts with an IP address, followed by the canonical name and optional aliases.

Verify the mapping:

```bash
getent hosts db01
```

Expected output:

```text
192.0.2.10      db01.example.internal db01
```

## 3. Inspect /etc/resolv.conf

Show the active resolver configuration:

```bash
cat /etc/resolv.conf
```

Expected output:

```text
nameserver 192.0.2.53
nameserver 8.8.8.8
search example.internal
```

On most current systems this file is generated automatically. It is a symlink to a runtime file managed by `systemd-resolved` or NetworkManager. Do not edit it directly: the next network event overwrites the change.

## 4. Query DNS Directly

Resolve a name with `dig`:

```bash
dig example.com
dig +short example.com
```

Query one specific nameserver:

```bash
dig @<NAMESERVER> example.com
```

Use `nslookup` or `host` for quick checks:

```bash
nslookup example.com
host example.com
```

Resolve through the full system path, including `/etc/hosts`:

```bash
getent hosts example.com
```

**Options:**
- `+short`: Prints only the answer records.
- `@<NAMESERVER>`: Sends the query to one server and bypasses `/etc/resolv.conf`.
- `getent hosts`: Uses the same lookup path as applications. If `getent` works but `dig` fails, `/etc/hosts` provides the answer.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Inspect the Resolver Status

Ubuntu and current Debian releases commonly use `systemd-resolved`:

```bash
resolvectl status
```

The output lists the DNS servers, search domains, and DNSSEC mode for each link.

## 2. Inspect Resolver Statistics

Show cache and query statistics:

```bash
resolvectl statistics
```

Check the effective configuration for one interface:

```bash
resolvectl status <INTERFACE>
```

## 3. Flush the Cache

Clear the local DNS cache after changing records:

```bash
sudo resolvectl flush-caches
```

Verify with a direct query:

```bash
dig +short example.com
```

## 4. Override DNS Servers at Runtime

Set DNS servers for one link temporarily:

```bash
sudo resolvectl dns <INTERFACE> <NAMESERVER>
```

Persist the change in the netplan configuration:

```yaml
network:
  version: 2
  ethernets:
    <INTERFACE>:
      dhcp4: true
      nameservers:
        addresses:
          - 192.0.2.53
          - 8.8.8.8
        search:
          - example.internal
```

Apply with `sudo netplan apply`. Runtime overrides are lost at reboot or when the link is renegotiated.

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Inspect the DNS Configuration

NetworkManager applies DNS settings from connection profiles:

```bash
nmcli device show <INTERFACE> | grep DNS
```

Expected output:

```text
IP4.DNS[1]:                             192.0.2.53
IP4.DNS[2]:                             8.8.8.8
```

Inspect the profile values:

```bash
nmcli connection show <CONNECTION> | grep -i dns
```

## 2. Set DNS Servers

Modify the connection profile:

```bash
sudo nmcli connection modify <CONNECTION> ipv4.dns "192.0.2.53 8.8.8.8"
sudo nmcli connection up <CONNECTION>
```

Ignore DNS servers received over DHCP:

```bash
sudo nmcli connection modify <CONNECTION> ipv4.ignore-auto-dns yes
sudo nmcli connection up <CONNECTION>
```

Add a search domain:

```bash
sudo nmcli connection modify <CONNECTION> ipv4.dns-search example.internal
```

**Options:**
- `ipv4.dns`: Sets the DNS servers as a space-separated list.
- `ipv4.ignore-auto-dns yes`: Discards DNS servers provided by DHCP.
- `ipv4.dns-search`: Sets the search domains for short names.

## 3. Understand /etc/resolv.conf Handling

NetworkManager rewrites `/etc/resolv.conf` through its DNS plugin. Verify the file after every change:

```bash
ls -l /etc/resolv.conf
cat /etc/resolv.conf
```

RHEL 9 can optionally use `systemd-resolved`. Identify the active manager before making changes:

```bash
systemctl is-active NetworkManager
systemctl is-active systemd-resolved
```

Edits to `/etc/resolv.conf` do not survive a reconnect while a manager is active.

---

# Part 3: Important Notes

## Resolution Flow

| Step | Source | Selected by | Persistent change |
|---|---|---|---|
| 1 | `/etc/hosts` | `files` in `/etc/nsswitch.conf` | Edit the file |
| 2 | Per-link DNS cache | `systemd-resolved` | Flush with `resolvectl flush-caches` |
| 3 | Nameservers in `/etc/resolv.conf` | `dns` in `/etc/nsswitch.conf` | netplan or `nmcli` |
| 4 | Search domains for short names | Resolver configuration | netplan `search` or `ipv4.dns-search` |

## Troubleshooting Flow

| Symptom | Command | Interpretation |
|---|---|---|
| One name fails everywhere | `dig +short <NAME> @<NAMESERVER>` | Missing or incorrect record in the zone |
| One name fails locally | `getent hosts <NAME>` | Wrong or missing `/etc/hosts` mapping |
| All names fail | `resolvectl status` or `nmcli device show <INTERFACE>` | No nameservers configured |
| All names fail, IP connectivity works | `cat /etc/resolv.conf` | Stale or missing resolver file |
| Names resolve slowly | `resolvectl statistics` | Cache misses or unreachable first nameserver |
| Stale answer after a DNS change | `sudo resolvectl flush-caches` | Record cached by systemd-resolved |
| Short names fail | Check the `search` entry in `/etc/resolv.conf` | Missing or wrong search domain |
| Works for a user, fails for a service | `getent hosts <NAME>` in the service context | Different namespace or container network |

## Command Reference

| Operation | Command |
|---|---|
| Show resolution order | `grep hosts /etc/nsswitch.conf` |
| Query DNS | `dig +short <NAME>` |
| Query one server | `dig @<NAMESERVER> <NAME>` |
| Test the system path | `getent hosts <NAME>` |
| Show resolver status | `resolvectl status` |
| Flush the cache | `sudo resolvectl flush-caches` |
| Show DNS servers (NetworkManager) | `nmcli device show <INTERFACE> \| grep DNS` |
| Set DNS servers (NetworkManager) | `nmcli connection modify <CONNECTION> ipv4.dns "<NAMESERVERS>"` |

## Security Guidelines

- Do not edit `/etc/resolv.conf` directly on systems where systemd-resolved or NetworkManager manages it. Configure DNS through netplan or `nmcli` instead.
- Prefer internal resolvers for internal names. Do not send internal hostnames to public DNS services.
- Use search domains to keep short names working instead of growing `/etc/hosts`.
- Keep `/etc/hosts` small. Large local files are hard to audit and drift between machines.
- Verify both the direct query (`dig`) and the system path (`getent hosts`) before concluding that DNS is broken.
- Flush the cache after changing DNS records and confirm the new answer with `dig +short`.
- Do not raise resolver timeouts or disable validation to mask intermittent failures. Fix the resolver instead.
- Use trusted or encrypted DNS infrastructure for production systems.
- Replace placeholders such as `<NAME>`, `<NAMESERVER>`, and `<CONNECTION>` with actual values. Do not include the angle brackets in commands.
