---
title: "process-management"
weight: 70
---
**Scope:** Debian-based and RHEL-based Linux distributions  
**Topics:** Process listing, signals, priority, background jobs, zombie diagnosis, and the /proc filesystem.

## 1. List Running Processes

List all processes of all users in BSD style:

```bash
ps aux
```

**Columns:**
- `USER`: Owner of the process.
- `PID`: Process ID.
- `%CPU` and `%MEM`: CPU and memory usage.
- `VSZ` and `RSS`: Virtual and resident memory in kibibytes.
- `TTY`: Controlling terminal. `?` means no terminal.
- `STAT`: Process state. `R` running, `S` sleeping, `Z` zombie, `T` stopped.
- `START`: Start time of the process.
- `TIME`: Accumulated CPU time.
- `COMMAND`: Executed command.

List all processes in System V style:

```bash
ps -ef
```

**Columns:**
- `UID`: Owner of the process.
- `PID` and `PPID`: Process ID and parent process ID.
- `C`: CPU utilization.
- `STIME`: Start time.
- `TTY`, `TIME`: Terminal and accumulated CPU time.
- `CMD`: Full command line.

Filter the output with `grep`:

```bash
ps aux | grep <PROCESS_NAME>
```

## 2. Monitor Processes in Real Time

Start the interactive monitor:

```bash
top
```

**Options:**
- `P`: Sorts by CPU usage.
- `M`: Sorts by memory usage.
- `k`: Prompts for a PID and sends a kill signal.
- `1`: Shows one line per CPU core.
- `q`: Quits.

Install `htop` for a more readable interface.

On Debian-based systems:

```bash
apt install htop
```

On RHEL-based systems:

```bash
dnf install htop
```

Run it:

```bash
htop
```

## 3. Find Processes by Name

Find PIDs by process name:

```bash
pgrep <PROCESS_NAME>
pidof <PROCESS_NAME>
```

Show the process name next to the PID:

```bash
pgrep -l <PROCESS_NAME>
```

Match against the full command line:

```bash
pgrep -a -f <PATTERN>
```

Expected output:

```text
1234 /usr/bin/myapp --config /etc/myapp/myapp.conf
```

**Options:**
- `-l`: Prints the process name next to each PID.
- `-a`: Prints the full command line.
- `-f`: Matches the pattern against the full command line instead of the process name.

## 4. Send Signals to Processes

Send the default signal (`SIGTERM`) by PID:

```bash
kill <PID>
```

Send a signal to every process with a given name:

```bash
killall <PROCESS_NAME>
```

Match processes by name or full command line:

```bash
pkill <PROCESS_NAME>
pkill -f <PATTERN>
```

Force a stuck process to stop:

```bash
kill -9 <PID>
kill -KILL <PID>
```

**Options:**
- `-9` or `-KILL`: Sends `SIGKILL`. The process cannot catch or ignore it.
- `-1` or `-HUP`: Sends `SIGHUP`. Many daemons reload their configuration.
- `-f`: Matches the pattern against the full command line.

Signal reference:

| Signal | Number | Default action | Typical use |
|---|---|---|---|
| `SIGTERM` | 15 | Terminates | Graceful shutdown. Default of `kill`. |
| `SIGKILL` | 9 | Terminates immediately | Force a stuck process to stop. Cannot be caught. |
| `SIGHUP` | 1 | Terminates | Reload configuration. Sent when the terminal closes. |
| `SIGINT` | 2 | Terminates | Interrupt from the keyboard (`Ctrl+C`). |
| `SIGSTOP` | 19 | Stops | Pause a process. Cannot be caught. |
| `SIGCONT` | 18 | Continues | Resume a stopped process. |

Prefer `SIGTERM` first. Give the process a chance to clean up before using `SIGKILL`.

## 5. Adjust Process Priority

Priority values range from `-20` (highest priority) to `19` (lowest priority).

Start a command with a modified priority:

```bash
nice -n <NICE_VALUE> <COMMAND>
```

Change the priority of a running process:

```bash
renice -n <NICE_VALUE> -p <PID>
```

**Options:**
- `-n <NICE_VALUE>`: Sets the niceness value. Range `-20` to `19`.
- `-p <PID>`: Applies the change to an existing process.

Only root can set negative nice values or raise the priority of a process. Verify the result:

```bash
ps -o pid,ni,comm -p <PID>
```

Expected output:

```text
  PID  NI COMMAND
 1234   5 myapp
```

## 6. Run Jobs in the Background

Start a job in the background:

```bash
<COMMAND> &
```

List the current shell jobs:

```bash
jobs
```

Suspend a foreground job with `Ctrl+Z`, then resume it:

```bash
bg %<JOB_ID>
```

Bring a job back to the foreground:

```bash
fg %<JOB_ID>
```

Keep a job running after logout:

```bash
nohup <COMMAND> > <LOG_FILE> 2>&1 &
```

**Options:**
- `&`: Starts the job in the background.
- `bg`: Resumes a stopped job in the background.
- `fg`: Brings a background or stopped job to the foreground.
- `nohup`: Ignores `SIGHUP` so the job survives logout. Without redirection, output goes to `nohup.out`.
- `Ctrl+C`: Sends `SIGINT` and terminates the foreground job.

Background jobs belong to the shell session. Use `nohup` or a systemd service for long-running work.

## 7. Diagnose Zombie Processes

List zombie processes:

```bash
ps aux | awk '$8 ~ /Z/'
```

Alternatively:

```bash
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /Z/'
```

Identify the parent process:

```bash
ps -o pid,ppid,stat,cmd -p <ZOMBIE_PID>
```

A zombie is a terminated process whose exit status has not been read by its parent. It consumes no CPU and almost no memory, but it retains a process table entry.

Ask the parent to reap its children:

```bash
kill -SIGCHLD <PPID>
```

If the parent does not respond, restart the parent service. When the parent exits, the zombie is adopted by init or systemd and removed automatically. `SIGKILL` cannot remove a zombie because the process is already dead.

## 8. Inspect the /proc Filesystem

Every running process has a directory under `/proc/<PID>`. Inspect it:

```bash
ls /proc/<PID>
```

Common entries:

- `/proc/<PID>/cmdline`: Command-line arguments, separated by null bytes.
- `/proc/<PID>/status`: State, memory, threads, and UID in readable form.
- `/proc/<PID>/exe`: Symlink to the executed binary. Resolve it with `readlink`.
- `/proc/<PID>/cwd`: Symlink to the working directory.
- `/proc/<PID>/fd`: Open file descriptors.
- `/proc/<PID>/environ`: Environment variables, separated by null bytes.

```bash
cat /proc/<PID>/cmdline
cat /proc/<PID>/status
readlink /proc/<PID>/exe
readlink /proc/<PID>/cwd
ls -l /proc/<PID>/fd
```

`/proc/<PID>` disappears when the process exits.

---

# Part 3: Important Notes

## ps Output Comparison

| Property | `ps aux` | `ps -ef` |
|---|---|---|
| Style | BSD | System V |
| Selects all processes | With `a` and `x` | With `-e` |
| Parent PID | Not shown | `PPID` column |
| Memory usage | `%MEM`, `VSZ`, `RSS` | Not shown |
| CPU usage | `%CPU` | `C` column |
| Common use | Resource consumption | Process hierarchy |

## Signal Delivery Tools

| Tool | Target | Example |
|---|---|---|
| `kill` | One PID or job | `kill <PID>` |
| `killall` | Every process with the exact name | `killall <PROCESS_NAME>` |
| `pkill` | Processes matching a name or pattern | `pkill -f <PATTERN>` |

## Job Control Reference

| Action | Command | Effect |
|---|---|---|
| Background | `<COMMAND> &` | Starts the job in the background |
| Suspend | `Ctrl+Z` | Stops the foreground job |
| Resume in background | `bg %<JOB_ID>` | Continues a stopped job |
| Resume in foreground | `fg %<JOB_ID>` | Brings the job forward |
| Survive logout | `nohup <COMMAND> &` | Ignores `SIGHUP` |

## Security Guidelines

- Replace placeholders such as `<PID>`, `<PROCESS_NAME>`, and `<NICE_VALUE>` with real values. Do not include the angle brackets in commands.
- Send `SIGTERM` before `SIGKILL`. Forced kills can leave incomplete writes and corrupted state.
- Verify PIDs with `ps` before sending signals. `killall` and `pkill` affect every matching process.
- Test `pkill -f <PATTERN>` with `pgrep -f <PATTERN>` first to preview the matched processes.
- Do not run application processes as root. Use dedicated system accounts.
- Only root can set negative nice values or signal processes owned by other users.
- Treat `/proc/<PID>/environ` and `/proc/<PID>/cmdline` as sensitive; they may contain credentials.
- Redirect `nohup` output to a log file with restrictive permissions instead of relying on `nohup.out`.
- Fix zombie accumulation in the parent process. Do not attempt to kill the zombie itself.
