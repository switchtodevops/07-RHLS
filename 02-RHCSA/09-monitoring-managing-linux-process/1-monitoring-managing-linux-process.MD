# Monitoring and Managing Linux Processes

This chapter is about finding running processes, understanding their states, controlling foreground and background jobs, sending signals, and checking system load. I’ve excluded the quiz, guided exercises, and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. What is a process?

A process is a running instance of a program. It has:

- A unique PID (process ID)
- A PPID (parent process ID)
- Allocated memory and system resources
- An owner and associated privileges
- One or more threads of program code
- A current state

When one process starts another, the new process is its child. The original is the parent. Linux starts system processes from systemd, which has PID 1.

When a child finishes, it releases its resources. Its parent then collects its remaining process-table entry. If that entry has not yet been collected, the child appears as a zombie.

## 2. Process states

The `S` column in `top` and the `STAT` column in `ps` show a process’s state.

| State | Meaning |
| --- | --- |
| `R` | Running on a CPU or ready and waiting for CPU time |
| `S` | Interruptible sleep; waiting for an event, resource, or signal |
| `D` | Uninterruptible sleep, usually waiting for critical I/O; ordinary signals cannot interrupt it |
| `T` | Stopped or being traced, such as after `Ctrl+Z` or `SIGSTOP` |
| `Z` | Zombie; the process has exited but its parent has not yet collected its entry |
| `I` | Idle kernel thread |

A process in `R` is not necessarily using the CPU at that exact moment; it may be waiting in the runnable queue. A process in `D` may contribute to load even when CPU use is low, because it is waiting for I/O.

## 3. List and inspect processes

### `ps`

`ps` displays a snapshot of processes. With no options, it normally shows processes belonging to your effective user ID and terminal.

Common commands:

```bash
ps
ps aux
ps lax
ps -ef
ps --forest
ps j
```

- `ps aux` lists all processes, including those without a controlling terminal.
- `ps -ef` is another common format for listing all processes.
- `ps lax` displays detailed process information.
- `ps --forest` shows parent and child relationships in a tree.
- `ps j` includes job, process-group, session, and terminal information.

Exam detail: `ps aux` and `ps -aux` are not equivalent. Use `ps aux` when you want the common BSD-style listing.

Useful `ps aux` columns:

- `USER`: process owner
- `PID`: process ID
- `%CPU`, `%MEM`: CPU and memory use
- `VSZ`: virtual memory size
- `RSS`: resident physical memory
- `TTY`: controlling terminal; `?` means no controlling terminal
- `STAT`: process state
- `START`, `TIME`: start time and accumulated CPU time
- `COMMAND`: command that started the process

To find a process by name:

```bash
ps aux | grep process_name
```

`grep` may show up in its own results. When possible, use `pgrep`:

```bash
pgrep -l process_name
```

### Process trees and sessions

```bash
pstree
pstree -p username
```

`pstree` shows parent/child relationships. `ps j` shows fields such as:

- `PID`: process ID
- `PPID`: parent process ID
- `PGID`: process-group ID
- `SID`: session ID
- `TTY`: controlling terminal

A job started in a terminal belongs to that terminal’s session. A system daemon usually has `?` in the `TTY` column and cannot be brought to the foreground from your shell.

## 4. Job control: foreground and background

A job is a command or pipeline started from a shell. It may contain one or more processes.

- The foreground job can read input from the terminal and receives keyboard-generated signals.
- A background job runs without occupying the shell prompt. It cannot read terminal input; if it tries, it may be stopped.

### Start a job in the background

Add `&` to the end of the command:

```bash
sleep 10000 &
```

The shell displays a job number and PID, for example:

```text
[1] 5947
```

### List shell jobs

```bash
jobs
```

Example:

```text
[1]+ Running    sleep 10000 &
```

The `+` marks the current default job; `-` marks the previous job.

### Bring a job to the foreground

```bash
fg %1
```

If you omit the job number, `fg` uses the current default job.

### Suspend a foreground job

Press:

```text
Ctrl+Z
```

This suspends the job and returns you to the shell prompt.

### Resume a suspended job in the background

```bash
bg %1
```

### Terminate a background job

```bash
kill %1
```

Here, `%1` is a shell job number. A PID is a process ID; a job number is specific to the current shell session.

## 5. Signals

A signal is a software notification sent to a process. Signals can come from the kernel, another process, or a keyboard shortcut.

Common signals:

| Signal | Usual number on x86_64 | Meaning |
| --- | --- | --- |
| `SIGHUP` | 1 | Terminal hangup; many services also use it to reload configuration |
| `SIGINT` | 2 | Interrupt, commonly sent by `Ctrl+C` |
| `SIGQUIT` | 3 | Quit; can generate a core dump |
| `SIGTERM` | 15 | Request termination and allow cleanup; default for `kill` |
| `SIGCONT` | 18 | Continue a stopped process |
| `SIGSTOP` | 19 | Stop a process; cannot be caught or ignored |
| `SIGTSTP` | 20 | Terminal stop, commonly sent by `Ctrl+Z` |
| `SIGKILL` | 9 | Force termination; cannot be caught, blocked, or ignored |

Signal numbers can differ on other hardware architectures. Prefer signal names when possible.

### Send a signal to a PID

```bash
kill PID
kill -TERM PID
kill -HUP PID
kill -SIGKILL PID
kill -9 PID
```

Despite its name, `kill` can send signals other than termination signals. See available signals with:

```bash
kill -l
```

Good practice: try `SIGTERM` first so the process can clean up. Use `SIGKILL` only if necessary; it stops the process immediately without allowing cleanup.

### Send signals by process name or selection

```bash
pgrep -l -u bob
pkill -u bob
pkill -SIGKILL -u bob
killall process_name
```

- `pgrep` finds matching process IDs; `pgrep -l` also lists process names.
- `pkill` sends a signal to processes matching selection criteria.
- `killall` sends a signal to processes matching a command name.
- Use `-u` to select processes owned by a user.
- `pkill -t tty3` selects processes associated with terminal `tty3`.
- `pkill -P PID` selects child processes of the specified parent PID.

Be careful with broad selections such as `pkill -u username`, because they affect multiple processes.

### Keyboard signals for the foreground job

| Key | Effect |
| --- | --- |
| `Ctrl+C` | Interrupt; sends `SIGINT` |
| `Ctrl+Z` | Suspend; sends `SIGTSTP` |
| `Ctrl+\` | Quit; sends `SIGQUIT` and may produce a core dump |

## 6. Inspect and end a user session

```bash
w
w -f
who
```

- `w` shows logged-in users and what they are doing, along with session details and load average.
- `who` gives basic information about logged-in users.
- TTY values such as `tty3` usually refer to a directly connected console; `pts/1` is typically a pseudo-terminal, such as a terminal window or remote login.

To end a particular session, first identify its terminal using `w`, then use elevated privileges:

```bash
sudo pkill -t tty3
```

You need root or suitable `sudo` privileges to terminate another user’s processes. A normal user can normally terminate only their own processes.

## 7. Monitor system load

### Load average

```bash
uptime
```

The three load averages represent the last 1, 5, and 15 minutes. They are running averages of processes that are:

- Ready to run or waiting for CPU time (`R`)
- Waiting for critical I/O (`D`)

So, a high load average does not always mean the CPU is busy. If the load is high while CPU use is low, investigate disk or network I/O.

Check the number of logical CPUs with:

```bash
lscpu
```

As a rough guide, compare load average with the number of logical CPUs. For a system with four logical CPUs, a load average of 4 means about one runnable task per CPU on average. A value above the CPU count can indicate queued work, but also consider I/O and the trend across the 1-, 5-, and 15-minute values.

## 8. Monitor processes in real time with `top`

```bash
top
```

Unlike `ps`, `top` refreshes continuously and lets you sort, filter, and manage processes interactively.

Useful columns:

| Column | Meaning |
| --- | --- |
| `PID` | Process ID |
| `USER` | Process owner |
| `PR` | Scheduling priority |
| `NI` | Nice value |
| `VIRT` | Virtual memory used or mapped by the process |
| `RES` | Resident physical memory |
| `SHR` | Shared memory |
| `S` | Process state |
| `%CPU`, `%MEM` | CPU and memory usage |
| `TIME+` | Accumulated CPU time |
| `COMMAND` | Process command |

Useful `top` keys:

| Key | Action |
| --- | --- |
| `h` | Show help |
| `l`, `t`, `m` | Toggle load, task/thread, and memory header lines |
| `1` | Toggle per-CPU display |
| `s` | Change refresh interval |
| `Shift+M` | Sort by memory use |
| `Shift+P` | Sort by CPU use |
| `u` or `Shift+U` | Filter by user |
| `k` | Send a signal to a PID |
| `r` | Change a process’s nice value |
| `Shift+H` | Toggle between processes and threads |
| `Shift+W` | Save the current display configuration |
| `q` | Quit |

## Exam task approach

When asked to find, monitor, or stop a process:

1. Find it: `pgrep -l name`, `ps aux | grep name`, or `ps -ef`.
2. Inspect it: check its PID, owner, state, and terminal.
3. For a shell job: use `jobs`; then `fg`, `bg`, or `kill %job_number`.
4. For a process PID: use `kill PID` to request termination.
5. If it will not stop: try a stronger signal only when appropriate.
6. For high system load: check `uptime`, `lscpu`, and `top`; remember that I/O waits can raise load too.
7. Verify the result: run `pgrep`, `ps`, or `jobs` again.

## Quick memory check

- `ps` = process snapshot; `top` = live process view.
- PID identifies a process; `%job` identifies a shell job.
- `Ctrl+Z` suspends; `bg` resumes it in the background; `fg` brings it forward.
- `SIGTERM` requests a clean stop; `SIGKILL` forces a stop.
- Load average counts runnable and uninterruptible-I/O processes.
