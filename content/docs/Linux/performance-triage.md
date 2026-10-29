---
title: "performance-triage"
weight: 210
---
**Scope:** Distro-independent performance triage on Linux servers  
**Topics:** Load average, CPU versus I/O wait, memory and swap pressure, disk I/O, culprit processes.

Performance triage answers one question first: is the system CPU-bound, memory-bound, or I/O-bound? Measure before changing anything, because tuning the wrong resource hides the real bottleneck. The workflow below uses tools available on Debian-based and RHEL-based systems alike.

## 1. Read the Load Average

Inspect the load average:

```bash
uptime
cat /proc/loadavg
```

Expected output:

```text
12:41:03 up 5 days,  3:12,  2 users,  load average: 4.12, 3.87, 2.95
```

Compare the values with the number of CPU cores:

```bash
nproc
```

The three values are the 1-, 5-, and 15-minute averages of runnable plus uninterruptible tasks. A load of `8.0` on a 4-core system means roughly twice as much work as the CPUs can run; the same value on a 16-core system is idle. A load average far above the core count with little CPU usage usually indicates processes stuck in I/O.

## 2. Distinguish CPU-Bound from I/O-Bound

Sample the system once per second:

```bash
vmstat 1 5
```

Expected output:

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 512340  84120 923456    0    0    12    40  320  610 22  4 72  2  0
 9  3      0 402112  84200 930120    0    0   180   220 1890 2410 68  9 15  8  0
```

**Options:**
- `r`: Runnable processes. Sustained values above the core count indicate CPU saturation.
- `b`: Processes blocked in uninterruptible sleep. Sustained values above zero indicate I/O problems.
- `us`, `sy`, `id`, `wa`, `st`: CPU time in user space, kernel space, idle, I/O wait, and stolen by a hypervisor.
- `wa` above roughly 20 percent with a high `b` column points to storage as the bottleneck.

## 3. Inspect CPU Usage

Start the interactive monitor:

```bash
top
```

Inspect per-CPU utilization:

```bash
mpstat -P ALL 1
```

Install the tool if it is missing:

```bash
apt install sysstat
dnf install sysstat
```

High `sys` time suggests syscall or lock contention. High `wa` confirms the CPU is idle while waiting for storage.

## 4. Inspect Memory and Swap Pressure

Inspect memory usage:

```bash
free -m
```

Expected output:

```text
              total        used        free      shared  buff/cache   available
Mem:           7936        6210         214          88        1512        1392
Swap:          2047        1980          67
```

**Options:**
- `free`: Memory not used at all. A small value is normal; the kernel caches aggressively.
- `buff/cache`: Reclaimable page cache and buffers. This memory is available on demand.
- `available`: Estimation of memory usable for new applications without swapping. Trust this column, not `free`.
- `Swap` used near its total size: Severe memory pressure.

Watch swap activity in `vmstat` using the `si` and `so` columns:

- `si` (swap in): Pages read from swap into RAM per second.
- `so` (swap out): Pages written from RAM to swap per second.

Sustained non-zero `si` and `so` indicate swap thrashing. The system spends its time moving pages instead of doing work.

## 5. Inspect Disk I/O

Inspect per-device I/O statistics:

```bash
iostat -x 1
```

Expected output:

```text
Device  r/s     w/s   rkB/s   wkB/s  await  aqu-sz  %util
sda    82.00  140.00 3280.00 5600.00  18.40    4.10  97.00
```

**Options:**
- `await`: Average time in milliseconds for an I/O request, including queueing. Above roughly 20 ms on local storage deserves attention.
- `aqu-sz`: Average queue length. Sustained values above 1 to 2 mean requests are queuing.
- `%util`: Percentage of time the device had at least one request in flight. Near 100 percent on spinning disks means saturation; on NVMe devices it can be misleading because requests run in parallel.
- `rkB/s`, `wkB/s`: Throughput per direction. Compare with the expected workload.

## 6. Find the Culprit Processes

Sort processes by CPU usage:

```bash
ps aux --sort=-%cpu | head
```

Sort processes by memory usage:

```bash
ps aux --sort=-%mem | head
```

Inspect one process in detail:

```bash
pidstat -u -d 1 -p <PID>
```

Sample a suspect process before killing or restarting it. A single process pegging one core is CPU-bound; many processes in `D` state point to storage or a hung device.

---

# Part 3: Important Notes

## Triage Decision Table

| Symptom | First command | What to look for |
|---|---|---|
| System feels slow | `uptime`, `cat /proc/loadavg` | Load average versus `nproc` |
| High load, low CPU usage | `vmstat 1` | `b` column, high `wa`, suspect I/O |
| High CPU usage | `mpstat -P ALL 1`, `ps aux --sort=-%cpu \| head` | `us` versus `sy`; hot process |
| Applications failing to start | `free -m` | `available`, swap usage |
| Constant disk activity | `iostat -x 1` | `await`, `aqu-sz`, `%util` |
| Everything freezes briefly | `vmstat 1` | `si`/`so` sustained non-zero (swap thrash) |
| One service is slow | `pidstat -u -d 1 -p <PID>` | CPU versus I/O time of the process |

## vmstat Column Reference

| Column | Meaning | Healthy signal |
|---|---|---|
| `r` | Runnable processes | At or below the core count |
| `b` | Blocked processes | Zero or near zero |
| `si`/`so` | Swap in and out per second | Zero after warm-up |
| `us`/`sy` | User and system CPU time | Together below 100 with idle time available |
| `wa` | I/O wait | Below roughly 20 percent |
| `st` | Stolen time (virtual machines) | Zero or near zero |

## Security Guidelines

- Measure before tuning. Record `vmstat`, `free`, and `iostat` output before changing kernel parameters or service limits.
- Distinguish CPU-bound from I/O-bound before acting; the correct fix differs completely.
- Watch for swap thrash: sustained `si` and `so` indicate memory exhaustion, not a slow disk.
- Do not kill processes you cannot identify. Trace the owner first with `ps -o user,pid,cmd -p <PID>`.
- Prefer restarting the responsible service over rebooting, so that the evidence in the logs survives.
- Keep monitoring data on the server; performance captures can reveal paths, users, and workload details.
- Re-measure after every change and keep the change only if the metric actually improves.
- Treat sudden load spikes as potential abuse until proven otherwise: check `last`, `ss -tulpn`, and running processes first.
- Do not raise swap as a substitute for adding memory; swap thrash degrades every workload on the host.
- Run heavy diagnostic batches (`iostat -x 1`, `vmstat 1`) one at a time to avoid adding load during an incident.