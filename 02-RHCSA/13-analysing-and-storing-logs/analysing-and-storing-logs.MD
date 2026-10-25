# Analyzing and Storing Logs

This chapter covers where RHEL logs events, how to search and interpret them, how to preserve journal entries across reboots, and how to keep system time accurate. I’ve excluded the quiz, guided exercises, and lab. The notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. System logging architecture

Logs help administrators audit activity and troubleshoot problems. RHEL uses two main services:

- `systemd-journald` collects messages from the kernel, early boot, daemon standard output and error, and syslog. It stores them in a structured, indexed binary journal.
- `rsyslog` reads syslog events from the journal, filters them according to its configuration, then writes them to text log files under `/var/log` or forwards them elsewhere.

In RHEL 10, local syslog messages are sent to `/dev/log`, a special socket read by `systemd-journald`. `rsyslog` reads the messages from the journal using its `imjournal` module.

By default, the journal is stored under `/run/log/journal`, which is memory-backed and cleared at shutdown. The text logs managed by `rsyslog` under `/var/log` persist across reboots.

### Common log files

| Log file | Typical content |
| --- | --- |
| `/var/log/messages` | Most general system messages |
| `/var/log/secure` | Authentication and security events |
| `/var/log/maillog` | Mail server events |
| `/var/log/cron` | Scheduled job events |
| `/var/log/boot.log` | Boot console messages and local7 messages |

Some applications keep their own logs. For example, Apache writes logs under a subdirectory of `/var/log`.

## 2. Syslog facilities and priorities

Each syslog message has a facility (the subsystem that generated it) and a priority (severity).

### Common facilities

| Facility | Meaning |
| --- | --- |
| `kern` | Kernel messages |
| `user` | User-level messages |
| `mail` | Mail system |
| `daemon` | System daemons |
| `auth` | Authentication and security |
| `syslog` | Internal syslog messages |
| `lpr` | Printer messages |
| `news` | Network news |
| `uucp` | UUCP protocol |
| `cron` | Scheduled job messages |
| `authpriv` | Private authorization messages |
| `ftp` | FTP messages |
| `local0`–`local7` | Custom local facilities |

### Priority levels

Lower numbers mean higher severity.

| Number | Priority | Meaning |
| --- | --- | --- |
| 0 | `emerg` | System unusable |
| 1 | `alert` | Immediate action required |
| 2 | `crit` | Critical condition |
| 3 | `err` | Error |
| 4 | `warning` | Warning |
| 5 | `notice` | Significant normal event |
| 6 | `info` | Informational |
| 7 | `debug` | Debugging detail |

## 3. How rsyslog rules work

Rsyslog rules are configured in `/etc/rsyslog.conf` and files ending in `.conf` under `/etc/rsyslog.d/`.

A rule has a filter followed by an action:

```text
facility.priority    action
```

For example:

```text
authpriv.*    /var/log/secure
```

This sends every priority from the `authpriv` facility to `/var/log/secure`.

A priority selector is a threshold: it matches messages at that priority and more severe levels. For example:

```text
authpriv.warning    /var/log/secure
```

matches `warning` and higher-severity messages, such as `err` and `crit`. A `*` matches all values. The `none` keyword excludes a facility from that rule.

Common default rules include:

```text
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                  /var/log/secure
mail.*                                      -/var/log/maillog
cron.*                                      /var/log/cron
*.emerg                                     :omusrmsg:*
local7.*                                    /var/log/boot.log
```

A message can match more than one rule and therefore appear in more than one log file. The `:omusrmsg:*` action sends emergency messages to the terminals of logged-in users.

## 4. Read and monitor text logs

To read a log file:

```bash
less /var/log/messages
tail /var/log/secure
```

Use `tail -f` to follow a file and display new lines as they are written:

```bash
tail -f /var/log/secure
```

A typical rsyslog entry contains:

```text
Mar 20 20:11:48 localhost sshd[1433]: Failed password for student ...
```

- `Mar 20 20:11:48`: timestamp
- `localhost`: host that sent the message
- `sshd[1433]`: program name and process ID
- Remaining text: event message

### Send a test message

`logger` sends a message to the logging system. By default, it uses `user.notice`.

```bash
logger "test message"
logger -p local7.notice "test message for boot.log"
```

Use `-p` to specify a facility and priority. This is useful for testing rsyslog rules.

## 5. Log rotation

The `logrotate` command rotates logs to limit their size. A systemd timer triggers it; it checks rotation requirements daily, while many logs are rotated weekly.

A rotated log is renamed with a date or sequence suffix, and a new log file is created. The relevant service is notified after rotation. Old rotations are eventually removed to reclaim space. Rotation can be configured to happen by time or when a log reaches a specified size.

## 6. Search the system journal with `journalctl`

Use `journalctl` to read the structured system journal:

```bash
journalctl
```

Root has full access; regular users may not be able to see all entries. Journal messages are highlighted by priority.

### Limit the output

```bash
journalctl -n 5
journalctl -f
```

- `-n 5`: show the last five entries. Without a number, `-n` shows the last ten.
- `-f`: follow new journal entries, similar to `tail -f`. Press `Ctrl+C` to exit.

### Filter by priority or service

```bash
journalctl -p err
journalctl -u sshd.service
```

`-p err` shows entries of priority `err` or higher severity. Priorities can be named or numbered. `-u` restricts output to a systemd unit.

### Filter by time

```bash
journalctl --since today
journalctl --since "-1 hour"
journalctl --since "2025-06-01 20:30" --until "2025-06-04 10:00"
```

Use quotes around date-time values containing spaces. The command accepts `today`, `yesterday`, and `tomorrow`. If you omit the time, it defaults to midnight; if you omit the date, it uses the current day.

### Show detailed journal fields

```bash
journalctl -o verbose
```

Verbose output shows journal metadata that can be used to create precise searches. Useful fields include:

- `_COMM`: command name
- `_EXE`: executable path
- `_PID`: process ID
- `_UID`: user ID
- `_SYSTEMD_UNIT`: systemd unit

You can filter by multiple fields:

```bash
journalctl _SYSTEMD_UNIT=sshd.service _PID=2188
```

## 7. Make the journal persistent

By default, journal entries under `/run/log/journal` disappear at reboot. To keep them across reboots, create `/var/log/journal` and flush the current journal:

```bash
mkdir -p /var/log/journal
journalctl --flush
```

With the default `Storage=auto` setting, the journal uses persistent storage when `/var/log/journal` exists. Otherwise, it uses volatile storage under `/run/log/journal`.

Check available boot records and view a specific boot:

```bash
journalctl --list-boots
journalctl -b
journalctl -b -1
```

- `-b`: current boot.
- `-b -1`: previous boot, useful when investigating a crash.
- `--list-boots`: list boots recorded in the journal. Positive numbers can select numbered boots too.

### Journal rotation and size limits

Persistent journals are still rotated; they are not kept forever. By default, journal rotation happens monthly. Journald also limits disk use and aims to keep free space available.

RHEL 10 does not install `/etc/systemd/journald.conf` by default. The provider defaults are in `/usr/lib/systemd/journald.conf`. Do not edit the provider file directly. To change settings, copy it to `/etc/systemd/journald.conf` and edit the copy, or use a `.conf` drop-in under `/etc/systemd/journald.conf.d/`.

Important settings include:

| Setting | Purpose |
| --- | --- |
| `SystemMaxUse`, `RuntimeMaxUse` | Maximum disk space for persistent or runtime journal data |
| `SystemMaxFileSize`, `RuntimeMaxFileSize` | Maximum size of an individual journal file before rotation |
| `SystemKeepFree`, `RuntimeKeepFree` | Minimum file-system space to leave free |

The course gives defaults of up to 10% of the file system for journal use, capped at 4 GB, and keeping at least 15% free. After changing journald configuration:

```bash
systemctl restart systemd-journald
```

Only files ending in `.journal` or `.journal~` are treated as journal files.

## 8. Set and synchronize system time

Accurate time is important for comparing logs across systems. NTP synchronizes system clocks with time sources. `timedatectl` displays the local and universal time, time zone, RTC time, and synchronization status:

```bash
timedatectl
```

### Configure the time zone

List available time zones:

```bash
timedatectl list-timezones
```

`tzselect` interactively suggests a time-zone name but does not change the system setting. Set the time zone as root with:

```bash
timedatectl set-timezone Europe/Amsterdam
timedatectl set-timezone UTC
```

Use IANA names in `Continent/City` form. UTC can be set directly with `timedatectl`.

### Set the time manually

```bash
timedatectl set-time "2026-10-25 14:30:00"
```

Manual time changes can fail while automatic synchronization is enabled. Turn NTP synchronization off first if necessary:

```bash
timedatectl set-ntp false
```

In RHEL, this setting controls the `chronyd` NTP service.

### Configure `chronyd` and NTP sources

`chronyd` synchronizes the system clock using configured NTP servers. It can also track RTC drift when network time is unavailable; the drift data is stored in `/var/lib/chrony/drift`.

An NTP source’s stratum indicates its distance in hops from a reference clock: the reference clock is stratum 0, a directly connected server is stratum 1, and a client of that server is stratum 2. Organizations often use public NTP sources or internal servers.

Configure sources in `/etc/chrony.conf`:

```text
pool 2.rhel.pool.ntp.org iburst
server ntp.example.com iburst
```

- `server` adds one server; it can be used multiple times.
- `pool` specifies a DNS name that can resolve to multiple servers whose addresses may change.
- `iburst` makes several quick measurements so `chronyd` can synchronize sooner.

RHEL normally includes a public pool source by default. In an isolated network, configure reachable internal NTP servers. The course recommends using several sources when possible for resilience and to help account for network delay.

After changing sources, restart `chronyd`:

```bash
systemctl restart chronyd
```

Check sources and synchronization:

```bash
chronyc sources -v
```

Common source-state markers:

| Marker | Meaning |
| --- | --- |
| `*` | Current best source |
| `+` | Also contributing to synchronization |
| `-` | Usable, but not currently selected |
| `x` | May be faulty |
| `~` | Time varies too much |
| `?` | Currently unusable or not yet validated |

A `Reach` value below `377` means fewer than the last eight polls succeeded. For full numeric IP addresses without DNS name resolution, use:

```bash
chronyc -n sources
```

The `-n` must come between `chronyc` and `sources`.

## Exam workflow

- For a recent error: `journalctl -p err --since today`
- For a service issue: `journalctl -u SERVICE --since "-1 hour"`
- For a previous boot: `journalctl --list-boots`, then `journalctl -b -1`
- To watch a text log: `tail -f /var/log/secure`
- To retain journal entries after reboot: create `/var/log/journal`, then run `journalctl --flush`
- To check or set time: `timedatectl`; use `timedatectl set-timezone ...` for the zone
- To verify NTP sources: `chronyc sources -v` and `timedatectl`

Keep the storage types straight: journal entries are structured binary records queried with `journalctl`; rsyslog files are text under `/var/log` and can be read with tools such as `less` and `tail`.
