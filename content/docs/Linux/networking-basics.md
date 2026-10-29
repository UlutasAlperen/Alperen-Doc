---
title: "networking-basics"
weight: 90
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Interface inspection, links, routes, neighbors, sockets, hostname configuration, netplan, NetworkManager, and verification.

## 1. Inspect Network Interfaces

List all interfaces and addresses:

```bash
ip addr show
```

Use the compact view:

```bash
ip -br addr
```

Expected output:

```text
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             192.0.2.10/24 fe80::a00:27ff:fe1a:2b3c/64
```

## 2. Inspect and Change the Link State

Show link state and MAC addresses:

```bash
ip link show
```

Bring an interface up:

```bash
sudo ip link set <INTERFACE> up
```

## 3. Inspect Routes

Show the routing table:

```bash
ip route show
```

Expected output:

```text
default via 192.0.2.1 dev eth0
192.0.2.0/24 dev eth0 proto kernel scope link src 192.0.2.10
```

Check which route a destination uses:

```bash
ip route get <TARGET_IP>
```

## 4. Inspect Neighbors

Show the ARP/NDP neighbor table:

```bash
ip neigh show
```

Expected output:

```text
192.0.2.1 dev eth0 lladdr 00:11:22:33:44:55 REACHABLE
```

## 5. Inspect Listening Ports

List listening TCP and UDP sockets with owning processes:

```bash
ss -tulpn
```

Expected output:

```text
Netid State  Local Address:Port  Process
tcp   LISTEN 0.0.0.0:22          users:(("sshd",pid=812,fd=3))
tcp   LISTEN 0.0.0.0:80          users:(("nginx",pid=950,fd=6))
```

**Options:**
- `-t`: Shows TCP sockets.
- `-u`: Shows UDP sockets.
- `-l`: Shows listening sockets only.
- `-p`: Shows the owning process. Requires root for processes of other users.
- `-n`: Shows numeric port numbers instead of service names.

## 6. Test Connectivity

Test reachability and latency:

```bash
ping -c 4 <GATEWAY_IP>
ping -c 4 <TARGET_IP>
```

**Options:**
- `-c 4`: Sends four packets and stops.

## 7. Inspect the Hostname

Show the hostname and related settings:

```bash
hostnamectl
```

Set a static hostname:

```bash
sudo hostnamectl set-hostname <HOSTNAME>
```

The static hostname is stored in `/etc/hostname` and is applied at boot on both distribution families.

---

# Part 1: Debian-Based Systems

**Examples:** Debian, Ubuntu

## 1. Identify the Netplan Configuration

List existing netplan files:

```bash
ls /etc/netplan/
```

Expected output:

```text
00-installer-config.yaml
```

File names end in `.yaml` and are applied in lexical order.

## 2. Write a Static Address Configuration

Edit the file:

```bash
sudo vim /etc/netplan/00-installer-config.yaml
```

Example configuration:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    <INTERFACE>:
      dhcp4: no
      addresses:
        - 192.0.2.10/24
      routes:
        - to: default
          via: 192.0.2.1
      nameservers:
        addresses:
          - 192.0.2.53
          - 8.8.8.8
```

For DHCP, set `dhcp4: true` and drop the `addresses`, `routes`, and `nameservers` keys.

## 3. Validate and Apply

Validate the syntax and generate the backend configuration:

```bash
sudo netplan generate
```

Apply with automatic rollback:

```bash
sudo netplan try
```

`netplan try` applies the configuration and rolls it back after 120 seconds unless it is confirmed. Use it for remote sessions.

Apply without rollback:

```bash
sudo netplan apply
```

## 4. Verify the Result

Check the address and the default route:

```bash
ip -br addr show <INTERFACE>
ip route show
```

Confirm the gateway is reachable:

```bash
ping -c 4 192.0.2.1
```

---

# Part 2: RHEL-Based Systems

**Examples:** RHEL, Rocky Linux, AlmaLinux

## 1. Inspect Devices and Connections

Show device states:

```bash
nmcli device status
```

Expected output:

```text
DEVICE  TYPE      STATE      CONNECTION
eth0    ethernet  connected  eth0
lo      loopback  unmanaged  --
```

Show connection profiles:

```bash
nmcli device show <INTERFACE>
nmcli connection show
```

## 2. Configure a Static Address

Modify the active connection profile:

```bash
sudo nmcli connection modify <CONNECTION> \
  ipv4.method manual \
  ipv4.addresses 192.0.2.10/24 \
  ipv4.gateway 192.0.2.1 \
  ipv4.dns "192.0.2.53 8.8.8.8"
```

Activate the changes:

```bash
sudo nmcli connection up <CONNECTION>
```

Switch back to DHCP:

```bash
sudo nmcli connection modify <CONNECTION> ipv4.method auto
sudo nmcli connection up <CONNECTION>
```

**Options:**
- `ipv4.method manual`: Configures a static address. Use `auto` for DHCP.
- `ipv4.addresses`: Sets the address and prefix length.
- `ipv4.gateway`: Sets the default gateway.
- `ipv4.dns`: Sets the DNS servers for this connection.

## 3. Verify the Result

Check the applied configuration:

```bash
ip -br addr show <INTERFACE>
ip route show
nmcli device show <INTERFACE>
```

Confirm that `IP4.ADDRESS` and `IP4.GATEWAY` contain the expected values.

---

# Part 3: Important Notes

## Legacy Interface Configuration

On older Debian systems, interfaces are configured in `/etc/network/interfaces` with the `ifupdown` tools. Example:

```text
auto eth0
iface eth0 inet static
    address 192.0.2.10/24
    gateway 192.0.2.1
```

Apply with `ifup eth0` or `systemctl restart networking`. On current releases, netplan is the supported configuration path. Do not mix `ifupdown` and netplan configuration for the same interface.

On RHEL 7 and earlier, interface files were stored in `/etc/sysconfig/network-scripts/`. RHEL 9 manages these profiles through NetworkManager. Prefer `nmcli` over editing keyfiles directly.

## Configuration File Locations

| Item | Debian-based | RHEL-based |
|---|---|---|
| Interface configuration | `/etc/netplan/*.yaml` | NetworkManager connection profiles |
| Connection profiles | Not used | `/etc/NetworkManager/system-connections/` |
| Legacy interface files | `/etc/network/interfaces` | `/etc/sysconfig/network-scripts/` |
| Static hostname | `/etc/hostname` | `/etc/hostname` |
| Local name mappings | `/etc/hosts` | `/etc/hosts` |

## Verification Steps

Run these checks after every configuration change:

1. Confirm the interface is up: `ip link show <INTERFACE>`.
2. Confirm the expected address: `ip -br addr show <INTERFACE>`.
3. Confirm the default route: `ip route show`.
4. Confirm the gateway is reachable: `ping -c 4 <GATEWAY_IP>`.
5. Confirm external connectivity: `ping -c 4 8.8.8.8`.
6. Confirm that name resolution works.

## Command Reference

| Operation | Debian-based | RHEL-based |
|---|---|---|
| Show addresses | `ip addr show` | `ip addr show` |
| Show routes | `ip route show` | `ip route show` |
| Show listening ports | `ss -tulpn` | `ss -tulpn` |
| Edit configuration | Text files in `/etc/netplan/` | `nmcli connection modify` |
| Apply configuration | `netplan apply` | `nmcli connection up <CONNECTION>` |
| Test before applying | `netplan try` | Not available |
| Set hostname | `hostnamectl set-hostname <HOSTNAME>` | `hostnamectl set-hostname <HOSTNAME>` |

## Security Guidelines

- Do not edit and apply network configuration blindly over SSH. Keep console access or use `netplan try` before `netplan apply`.
- Replace placeholders such as `<INTERFACE>`, `<CONNECTION>`, and `<HOSTNAME>` with actual values. Do not include the angle brackets in commands.
- Make one change at a time and verify connectivity after each change.
- Persist configuration through netplan or NetworkManager. Do not leave long-lived addresses configured only with `ip addr add`.
- Document static addresses centrally and avoid conflicts with active DHCP ranges.
- Restrict management access to dedicated networks where possible. Do not expose management ports on public interfaces.
- Keep an out-of-band or console path available before changing network settings on a remote host.
