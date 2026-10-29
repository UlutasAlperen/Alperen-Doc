---
title: "process-internals"
weight: 75
---
**Scope:** Distro-independent Linux internals: programs, processes, and namespaces  
**Topics:** Programs versus processes, fork and exec, process tree, /proc mapping, namespaces, and containers.

## 1. Recognize That a Program Is a File

A program is an ordinary file on disk: either a compiled binary in ELF format or a script interpreted through a shebang line. The execute permission turns a file into a runnable program.

Identify the type of a binary:

```bash
file /usr/bin/ls
```

Expected output:

```text
/usr/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2
```

Inspect the first line of a script to see its interpreter:

```bash
head -1 <SCRIPT>
```

Expected output:

```text
#!/bin/bash
```

Check the permission bits:

```bash
ls -l /usr/bin/ls
```

Expected output:

```text
-rwxr-xr-x 1 root root 142024 Feb  7  2024 /usr/bin/ls
```

The `x` marks the file as executable. Without it the kernel refuses to run the file. The shebang (`#!`) tells the kernel which interpreter to load for a script.

## 2. Load the File as a Process

When the file is executed, the kernel loads it into memory, assigns a PID, and allocates memory, file descriptors, and other resources. The running instance is a process. The file on disk stays unchanged.

Show the file behind the current shell process:

```bash
readlink /proc/$$/exe
```

`$$` expands to the PID of the current shell.

Expected output:

```text
/usr/bin/bash
```

`/proc/self` always points to the process that reads it, so `readlink /proc/self/exe` resolves to the tool doing the reading.

The same file can back many processes at once. Each process owns its own PID and resources; they share nothing but the executable file.

## 3. Trace the fork/exec Model

Every new process is created as a child of an existing process: the parent forks a copy of itself, the child execs the new program. The kernel records the relationship as PID and PPID.

Show the current shell's PID:

```bash
echo $$
```

Run a child process and inspect the pair:

```bash
ps -o pid,ppid,comm
```

Expected output:

```text
  PID  PPID COMMAND
  4321  4100 bash
  4567  4321 ps
```

The shell (`bash`, PID 4321) is the parent of `ps` (PID 4567). The child's `PPID` equals the parent's `PID`.

What happens when the shell runs `ls`:

1. The shell calls `fork()` and the kernel creates a child process as a copy of the shell.
2. The child calls `exec()` and its memory image is replaced by `/usr/bin/ls`.
3. The kernel schedules the child and `ls` runs.
4. `ls` exits and reports its exit status.
5. The parent calls `wait()`, collects the exit status, and the kernel removes the child entry. This last step is called reaping.

If the parent does not reap the child, the child remains as a zombie. `fork` creates the process; `exec` gives it its program.

## 4. Inspect the Process Tree

Display the process tree with PIDs:

```bash
pstree -p
```

Expected output:

```text
systemd(1)─┬─sshd(812)───sshd(1234)───bash(1523)
            ├─cron(760)
            └─systemd-journald(645)
```

Show an indented tree with `ps`:

```bash
ps -e --forest -o pid,ppid,comm
```

Nearly every process descends from PID 1 (`systemd`). When a parent exits before its children, the orphaned processes are reparented to PID 1 or to the nearest subreaper, which then reaps them.

After a fork, the child initially shares the parent's memory pages. The kernel copies a page only when one side writes to it. This copy-on-write behavior makes forking cheap; a full memory copy almost never happens.

## 5. Map the Process Back to Files

Find the PID of a running program:

```bash
pgrep <PROCESS_NAME>
```

Show the file the process runs:

```bash
readlink /proc/<PID>/exe
```

Expected output:

```text
/usr/bin/myapp
```

Show the working directory and the open files:

```bash
readlink /proc/<PID>/cwd
ls -l /proc/<PID>/fd
```

A process exists only while it runs: its `/proc/<PID>` directory disappears when it exits. The file exists permanently until it is deleted. Deleting the file does not stop a running process; the kernel keeps the inode alive until the last reference is closed.

## 6. Restrict Processes with Namespaces

A namespace gives a process its own isolated view of a system resource. Processes in the same namespace share that view; processes in different namespaces cannot see each other's resources.

Linux provides seven namespaces:

- `PID`: Isolates process IDs. The first process inside appears as PID 1.
- `NET`: Isolates network interfaces, routes, and ports.
- `MNT`: Isolates mount points and the visible filesystem layout.
- `UTS`: Isolates the hostname and domain name.
- `IPC`: Isolates shared memory, semaphores, and message queues.
- `USER`: Isolates user and group IDs and maps them to different IDs.
- `CGROUP`: Isolates the view of the cgroup hierarchy.

List the namespaces on the system:

```bash
lsns
```

Expected output:

```text
        NS TYPE   NPROCS   PID USER COMMAND
4026531835 cgroup    180     1 root /sbin/init
4026531840 mnt       175     1 root /sbin/init
4026531860 net       180     1 root /sbin/init
```

### Create a PID namespace

Run as root:

```bash
unshare --pid --fork --mount-proc bash
```

Inspect the processes inside:

```bash
ps -o pid,ppid,comm
```

Expected output:

```text
  PID  PPID COMMAND
    1     0 bash
    7     1 ps
```

Only processes started inside the namespace are visible. The `bash` process is PID 1 inside the namespace even though it has a much larger PID on the host. Leave the namespace with `exit`.

**Options:**
- `--pid`: Creates a new PID namespace.
- `--fork`: Forks a child that becomes PID 1 inside the namespace. Required with `--pid`.
- `--mount-proc`: Mounts a fresh procfs so `ps` reports namespace PIDs instead of host PIDs.
- `--net`: Creates a new network namespace.
- `--mount`: Creates a new mount namespace.

### Create a network namespace

Run as root:

```bash
unshare --net bash
```

Inspect the interfaces:

```bash
ip addr
```

Expected output:

```text
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```

Only the loopback interface exists, and it is down. Host interfaces are not visible from inside the namespace.

### Enter an existing namespace

Run a command inside the namespaces of a running process:

```bash
nsenter -t <PID> -n <COMMAND>
```

For example, inspect the network namespace of PID 1234:

```bash
nsenter -t 1234 -n ip addr
```

Show the namespace links of a process:

```bash
ls -l /proc/<PID>/ns
```

Each entry is a symlink whose target identifies the namespace instance. Two processes with the same target share that namespace.

## 7. Understand Containers as Restricted Processes

A container is an ordinary process confined by three mechanisms:

- Namespaces provide isolation: the process sees its own PIDs, mounts, network, and users.
- cgroups provide resource limits: CPU time, memory, and I/O are capped.
- A layered filesystem such as overlayfs provides the root filesystem, built from stacked read-only layers plus a writable top layer.

There is no special container process type. `ps` on the host lists container processes like any other.

Remove the PID isolation as a counterexample:

```bash
docker run --rm --pid=host alpine ps
```

Expected output:

```text
PID   USER     TIME  COMMAND
    1 root      0:02 /sbin/init
  812 root      0:00 /usr/sbin/sshd
1523  root      0:00 bash
```

The container now sees every host process. The same applies to `--net=host`, `--uts=host`, and `--ipc=host`: each flag removes one layer of isolation.

A Kubernetes pod is a group of containers that share one network namespace: the containers in a pod see the same interfaces and reach each other through `localhost`.

---

# Part 3: Important Notes

## Namespace Reference

| Namespace | Isolates | Effect when restricted | Inspect with |
|---|---|---|---|
| `PID` | Process IDs | The process sees its own PID numbering, starting at 1 | `lsns -t pid` |
| `NET` | Network stack | Interfaces, routes, and ports are private | `lsns -t net`, `ip addr` |
| `MNT` | Mount points | Mounts are invisible to other namespaces | `lsns -t mnt` |
| `UTS` | Hostname and domain name | `hostname` changes affect only the namespace | `lsns -t uts` |
| `IPC` | Shared memory, semaphores, message queues | IPC objects are private to the namespace | `lsns -t ipc` |
| `USER` | User and group IDs | A UID maps to another UID (root inside, unprivileged outside) | `lsns -t user` |
| `CGROUP` | cgroup root directory | The process sees its own cgroup hierarchy | `lsns -t cgroup` |

## Process Lifecycle

| Stage | What happens | Observed via |
|---|---|---|
| File on disk | The program exists as an executable file | `file`, `ls -l` |
| `fork` | The parent creates a child copy (copy-on-write) | A new PID appears in `ps` |
| `exec` | The child's image is replaced by the program | `/proc/<PID>/exe` points to the new file |
| Running | The kernel schedules the process and allocates resources | `top`, `/proc/<PID>/` |
| Exit | The process terminates and becomes a zombie | `STAT` column shows `Z` |
| Reap | The parent reads the exit status and the entry is removed | The PID disappears from `ps` |

## Namespace Tools

| Tool | Purpose | Example |
|---|---|---|
| `lsns` | Lists namespaces and their member processes | `lsns -t pid` |
| `unshare` | Runs a command in fresh namespaces | `unshare --pid --fork --mount-proc bash` |
| `nsenter` | Enters the namespaces of an existing process | `nsenter -t <PID> -n <COMMAND>` |

## Security Guidelines

- Replace placeholders such as `<PID>`, `<SCRIPT>`, and `<COMMAND>` with real values. Do not include the angle brackets in commands.
- Run untrusted workloads inside namespaces. Do not execute unverified code in the host namespace.
- Avoid `--pid=host`, `--net=host`, and `--ipc=host` unless the workload requires them. Each flag removes one layer of isolation.
- Remember that PID 1 has special duties: it must reap orphaned children and handle signals that other processes can ignore.
- Treat namespaces as isolation, not as a complete security boundary. Combine them with cgroups, seccomp, and LSMs such as SELinux or AppArmor.
- Creating namespaces and entering foreign namespaces generally requires root or the `CAP_SYS_ADMIN` capability.
- Inspect `/proc/<PID>/ns` before debugging a container to confirm which namespace instance a process actually runs in.
- Verify the effect of `unshare` with `ps` or `ip addr` from inside the namespace before trusting the isolation.
- Keep container images minimal. A smaller root filesystem limits the tools available to a compromised process.
