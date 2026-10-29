---
title: "network-troubleshooting"
weight: 120
---
**Scope:** Distro-independent network troubleshooting on Debian-based and RHEL-based systems  
**Topics:** Layered diagnosis, ip, ss, ping, traceroute, mtr, dig, curl, tcpdump, packet capture, and error interpretation.

## 1. Work Through the Layers

Diagnose in a fixed order: link, address, route, DNS, port, then application. Stop at the first failing layer and fix it before continuing. Skipping layers produces false conclusions and wasted time.

## 2. Check the Link State

Inspect the interface:

```bash
ip link show <INTERFACE>
```

Expected output fragment:

```text
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ... state UP
```

`LOWER_UP` confirms a physical or virtual link. If it is missing, verify the cable, switch port, or hypervisor attachment. Check the carrier directly:

```bash
ethtool <INTERFACE> | grep "Link detected"
```

Bring the interface up if it is administratively down:

```bash
sudo ip link set <INTERFACE> up
```

## 3. Check the Address

Inspect the assigned addresses:

```bash
ip -br addr show <INTERFACE>
```

An interface without a `scope global` address cannot reach external networks. Verify DHCP operation or the static configuration, then renew or reapply it.

## 4. Check the Route

Inspect the routing table:

```bash
ip route show
```

Check the route selected for one destination:

```bash
ip route get <TARGET_IP>
```

Expected output:

```text
<TARGET_IP> via 192.0.2.1 dev eth0 src 192.0.2.10 uid 0
```

External targets require a default route. The `src` address must match the interface that should initiate the connection.

## 5. Check the Neighbor Table

Inspect the ARP/NDP table:

```bash
ip neigh show
```

The gateway should be `REACHABLE`, `STALE`, or `DELAY` after a ping. A persistent `FAILED` entry indicates a layer 2 problem between the host and the gateway.

## 6. Test Reachability

```bash
ping -c 4 <GATEWAY_IP>
ping -c 4 <TARGET_IP>
```

If the gateway answers but the target does not, the fault lies upstream or in routing. If both fail, check the link, address, and local firewall first.

## 7. Trace the Path

```bash
traceroute <TARGET_IP>
mtr -rw -c 20 <TARGET_IP>
```

`mtr` combines `ping` and `traceroute` and reports per-hop loss and latency. Install the tools if they are missing:

```bash
apt install mtr-tiny traceroute
dnf install mtr traceroute
```

## 8. Test DNS Separately

Resolve the name and compare it with the address used:

```bash
dig +short <NAME>
getent hosts <NAME>
ping -c 4 <NAME>
```

If pinging the IP works but pinging the name fails, the problem is name resolution, not connectivity. Test one server directly to isolate the resolver:

```bash
dig @<NAMESERVER> <NAME>
```

## 9. Inspect Listening Ports

Confirm that the service listens on the expected socket:

```bash
ss -tulpn
```

`0.0.0.0:80` accepts connections on all interfaces. `127.0.0.1:80` accepts local connections only. Inspect established connections:

```bash
ss -tanp
```

## 10. Test the Application Directly

Send an actual request:

```bash
curl -v http://<HOST>:<PORT>
curl -I https://<HOST>
```

**Options:**
- `-v`: Shows the full request and response exchange, including TLS negotiation.
- `-I`: Sends a HEAD request and prints the response headers.
- `--connect-timeout 5`: Fails fast when the connection cannot be established.

## 11. Capture Traffic

Capture traffic for one host:

```bash
sudo tcpdump -i <INTERFACE> -n host <TARGET_IP>
```

Capture DNS queries:

```bash
sudo tcpdump -i any -n port 53
```

Capture a limited sample of one service and write it to a file:

```bash
sudo tcpdump -i <INTERFACE> -n -c 100 -w /tmp/capture.pcap port <PORT>
```

**Options:**
- `-i <INTERFACE>`: Selects the interface. `any` captures on all interfaces.
- `-n`: Disables name resolution. Use it when DNS is the suspected problem.
- `host <IP>` and `port <PORT>`: Filter the capture. Always filter on busy hosts.
- `-c 100`: Stops after 100 packets.
- `-w <FILE>`: Writes a pcap file that can be opened in Wireshark.

## 12. Interpret Common Errors

| Error message | Meaning | Likely causes | Next command |
|---|---|---|---|
| Connection refused | The host answered with a TCP RST | Service stopped, wrong port, local REJECT rule | `ss -tulpn` |
| Connection timed out | No answer was received | Firewall DROP rule, wrong route, host down | `tcpdump`, firewall status |
| No route to host | ICMP host unreachable received | Missing route, stale route entry, REJECT rule | `ip route get <TARGET_IP>` |
| Network is unreachable | No usable route on the source host | Link down, no default route | `ip route show` |
| Name resolution failed | The name could not be resolved | No nameservers, stale cache, NXDOMAIN | `dig +short <NAME>` |
| Connection reset by peer | The peer closed the connection abruptly | Service crash, TLS mismatch, proxy timeout | `curl -v`, application logs |

---

# Part 3: Important Notes

## Layered Checklist

| Layer | Check | Command |
|---|---|---|
| Link | Interface present and carrier detected | `ip link show <INTERFACE>` |
| Address | Expected address and prefix assigned | `ip -br addr show <INTERFACE>` |
| Route | Default route and route to target present | `ip route show`, `ip route get <TARGET_IP>` |
| Neighbor | Gateway resolvable at layer 2 | `ip neigh show` |
| Reachability | Target answers ICMP | `ping -c 4 <TARGET_IP>` |
| Path | Intermediate hops respond | `mtr -rw -c 20 <TARGET_IP>` |
| DNS | Name maps to the expected address | `dig +short <NAME>`, `getent hosts <NAME>` |
| Port | Service listens on the expected socket | `ss -tulpn` |
| Application | Service answers correctly | `curl -v http://<HOST>:<PORT>` |
| Packets | Actual traffic on the wire | `tcpdump -i <INTERFACE> -n host <TARGET_IP>` |

## Command Reference

| Tool | Purpose | Example |
|---|---|---|
| `ip` | Addresses, links, routes, neighbors | `ip route get <TARGET_IP>` |
| `ss` | Sockets and listening ports | `ss -tulpn` |
| `ping` | ICMP reachability and latency | `ping -c 4 <GATEWAY_IP>` |
| `traceroute` | Hop-by-hop path | `traceroute <TARGET_IP>` |
| `mtr` | Path with per-hop statistics | `mtr -rw -c 20 <TARGET_IP>` |
| `dig` | DNS queries | `dig +short example.com` |
| `getent` | System resolution path | `getent hosts <NAME>` |
| `curl` | HTTP and TLS testing | `curl -v https://<HOST>` |
| `tcpdump` | Packet capture | `tcpdump -i any -n port 53` |

## Security Guidelines

- Filter every capture with `host` or `port` and limit the packet count. Unfiltered captures collect unrelated traffic and consume disk space.
- Store capture files securely and delete them after analysis. pcap files can contain credentials from unencrypted protocols.
- Check the firewall before blaming DNS. Dropped packets and failed lookups often produce similar symptoms.
- Use `-n` with `ss`, `tcpdump`, and `ip` when name resolution is suspected. Otherwise the tools depend on the broken component.
- Verify reachability by IP first, then by name, to separate connectivity problems from resolution problems.
- Collect evidence before restarting services or changing configuration. Preserve logs and captures of the failing state.
- Avoid capturing traffic on interfaces that carry sensitive data unless the analysis requires it.
- Keep diagnosis read-only. Apply configuration changes in a controlled step after the failing layer is identified.
