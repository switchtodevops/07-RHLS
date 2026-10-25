# Scheduling System Tasks

This chapter covers three ways RHEL schedules recurring work: systemd timers, systemd-tmpfiles for temporary files, and cron. I’ve skipped the quiz, guided exercises, and lab. These notes follow the Chapter 12 teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. systemd timer units

A systemd timer unit is a file ending in `.timer`. It schedules another unit—usually a `.service`—to run at a specified time or interval. By default, the service has the same name as the timer: `job.timer` triggers `job.service`.

Timer events are recorded in the system journal, which helps with troubleshooting. In RHEL 10, systemd timers handle many system tasks traditionally scheduled through cron.

### List timers

```bash
systemctl list-units --type=timer
systemctl list-units --type=timer --all
```

The first command lists timers loaded in memory that are active, pending, or failed. `--all` includes inactive timers.

To see installed timer unit files and whether they are enabled:

```bash
systemctl list-unit-files --type=timer
```

The columns mean:

- `list-units`: `UNIT`, `LOAD`, `ACTIVE`, `SUB`, and `DESCRIPTION`
- `list-unit-files`: `UNIT FILE`, `STATE`, and `PRESET`

`PRESET` indicates whether a unit is enabled by default.

### Start, stop, enable, and disable timers

```bash
systemctl start job.timer
systemctl stop job.timer
systemctl enable job.timer
systemctl enable --now job.timer
systemctl disable job.timer
systemctl disable --now job.timer
```

- `start` and `stop` affect the current system session.
- `enable` and `disable` affect whether the timer starts automatically at boot.
- `enable` alone does not start it now.
- `disable` alone does not stop it now.
- `--now` combines the boot-time change with starting or stopping it immediately.

### Check a timer

```bash
systemctl status job.timer
```

The status can show whether the timer is waiting, its next `Trigger` time, and which service it `Triggers`.

### Timer schedule settings

Timer-specific options are in the `[Timer]` section of the unit file.

Calendar schedule:

```ini
[Timer]
OnCalendar=*:00/10
```

This triggers the paired service every ten minutes.

Calendar schedules can also specify dates and times. For example, `OnCalendar=2025-07-* 12:35,37,39:16` triggers at 12:35:16, 12:37:16, and 12:39:16 on each day of July 2025.

Relative interval:

```ini
OnUnitActiveSec=15min
```

This triggers the paired service 15 minutes after its last activation.

### Safely edit a system timer

Do not edit provider unit files under `/usr/lib/systemd/system/` directly: package updates can overwrite your changes. Copy the unit file to `/etc/systemd/system/` and edit that copy. The `/etc` version takes precedence over the provider version.

After editing:

```bash
systemctl daemon-reload
systemctl enable --now job.timer
```

Use `systemctl status job.timer` to check its state and next trigger.

## 2. Manage temporary files with systemd-tmpfiles

Applications use temporary files and directories for working data. Some locations, such as directories under `/run`, are memory-based and are cleared on reboot. Temporary data on persistent storage may need cleanup to prevent stale files and disk space problems.

The `systemd-tmpfiles` tool reads configuration files that define files and directories to create, remove, or set permissions and ownership for.

At boot, `systemd-tmpfiles-setup.service` runs `systemd-tmpfiles` and reads configuration from:

```text
/usr/lib/tmpfiles.d/*.conf
/run/tmpfiles.d/*.conf
/etc/tmpfiles.d/*.conf
```

The `systemd-tmpfiles-clean.timer` periodically triggers `systemd-tmpfiles-clean.service`, which runs:

```bash
systemd-tmpfiles --clean
```

Inspect the timer’s configuration and state with:

```bash
systemctl cat systemd-tmpfiles-clean.timer
systemctl status systemd-tmpfiles-clean.timer
```

The timer can use settings such as:

```ini
[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
```

Here, the first cleanup is scheduled 15 minutes after boot; later cleanups are scheduled 24 hours after the service was last activated.

### tmpfiles configuration format

Each configuration line has these fields:

```text
Type Path Mode UID GID Age Argument
```

Examples:

```text
d /run/systemd/seats 0755 root root -
D /home/student 0700 student student 1d
L /run/fstablink - root root - /etc/fstab
```

- `d`: create the directory if it does not exist; otherwise leave it in place.
- `D`: create the directory if needed and remove its contents when cleanup runs, according to the configured age.
- `L`: create a symbolic link. The final field specifies the link target.
- `Z`: recursively restore SELinux contexts, permissions, and ownership.
- `Age`: for cleanup rules, the age determines how old a file must be to qualify for removal.
- A hyphen (`-`) means no value is specified for that field.

For example:

```text
D /home/student 0700 student student 1d
```

This defines `/home/student` with mode `0700`, owned by `student:student`, and removes files older than one day when cleanup runs.

You can run the cleanup operation manually:

```bash
systemd-tmpfiles --clean
```

### tmpfiles configuration precedence

- `/etc/tmpfiles.d/`: administrator configuration and overrides.
- `/run/tmpfiles.d/`: temporary runtime configuration, usually managed by daemons.
- `/usr/lib/tmpfiles.d/`: package-provided configuration. Do not edit these files directly.

When files with the same name exist in these locations, the higher-precedence configuration is used: `/etc` overrides `/run`, which overrides `/usr/lib`. To customize a package-provided rule, copy its file into `/etc/tmpfiles.d/` and edit the copy.

When testing a new or modified configuration, apply only the commands from that single file by specifying the file on the `systemd-tmpfiles` command line.

## 3. Schedule system jobs with cron

Cron runs recurring commands at specified times. The course recommends using system-wide crontab files for system administration jobs, rather than scheduling system tasks under a user account.

### Cron schedule fields

A system-wide crontab entry has five time fields, then the user to run the command as, then the command:

```text
minute hour day-of-month month day-of-week user command
```

| Field | Values |
| --- | --- |
| Minute | 0–59 |
| Hour | 0–23 |
| Day of month | 1–31 |
| Month | 1–12 or month names |
| Day of week | 0–6 (Sunday is 0 or 7) or day names |

A `*` means every permitted value in that field. `*/2` means every second value; for example, in the minute field it means every two minutes.

Example:

```cron
*/2 * * * * root date >> /tmp/timestamp_example.txt
```

This runs `date` as root every two minutes and appends the output to the file.

### Where to put system cron jobs

Create custom system cron files in:

```text
/etc/cron.d/
```

Do not edit `/etc/crontab` directly; the course recommends using it as a syntax reference. Use a distinctive file name in `/etc/cron.d/` to reduce the chance of a naming conflict with a package-provided file.

Cron jobs may also be shell scripts placed in these directories:

```text
/etc/cron.hourly/
/etc/cron.daily/
/etc/cron.weekly/
/etc/cron.monthly/
```

These directories contain executable scripts, not crontab files. Make a script executable with:

```bash
chmod +x script_name
```

The `/etc/cron.d/0hourly` file uses `run-parts` to execute scripts in `/etc/cron.hourly` as root at one minute past each hour. `run-parts` runs the executable files it finds in the specified directory.

A system crontab file can set environment variables before its schedule entries:

```cron
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
```

The system-wide format includes the user field. Remember that this extra field belongs in system cron files; it is not part of the standard five-field user crontab format.

## 4. Anacron: run periodic jobs after downtime

Cron suits time-sensitive jobs on systems expected to be running at the scheduled time. Anacron helps make sure daily, weekly, or monthly jobs are not skipped just because a system was off or hibernating.

Its configuration is in `/etc/anacrontab`. It uses a different format from regular cron:

```text
period-in-days delay-in-minutes job-identifier command
```

Example entries:

```cron
1 5 cron.daily nice run-parts /etc/cron.daily
7 25 cron.weekly nice run-parts /etc/cron.weekly
@monthly 45 cron.monthly nice run-parts /etc/cron.monthly
```

- **Period:** how often the job runs, in days. `@daily` is equivalent to `1`; `@weekly` to `7`.
- **Delay:** how many minutes to wait before starting the job.
- **Job identifier:** unique name used in log messages.
- **Command:** the command to execute.

Other settings can appear in `/etc/anacrontab`:

- `RANDOM_DELAY` sets the maximum random delay added to jobs.
- `START_HOURS_RANGE` restricts the hours when jobs may start. If the job cannot run in that range on a given day, it must wait until the next day.

Files in `/var/spool/anacron` record when periodic jobs last ran. Cron and Anacron work together: the hourly cron job runs a script that can call `anacron -s`, which then processes the daily, weekly, and monthly jobs.

## Exam reminders

- `systemctl list-units` shows loaded timers; `systemctl list-unit-files` shows installed timer files and their enablement state.
- A timer usually triggers a same-named service.
- `enable` does not mean “start now”; use `enable --now` when both are required.
- For timer edits, work with a copy under `/etc/systemd/system/`, then run `systemctl daemon-reload`.
- For system cron, put custom entries in `/etc/cron.d/` and include the user field.
- Cron schedule files use five time fields; `/etc/anacrontab` uses period, delay, identifier, and command.
- Cron can miss a scheduled run when the system is off; Anacron is designed to catch up periodic jobs after downtime.
