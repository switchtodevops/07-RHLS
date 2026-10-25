# Controlling Services and Daemons

This chapter covers how Linux starts services, how systemd organizes them, and how to inspect and control them with `systemctl`. I’ve left out the quiz, guided exercises, and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. Services, daemons, and systemd

A daemon is a background process that performs a task. Its name often ends in `d`, such as `httpd` or `chronyd`.

A service is the systemd-managed unit for a system service. It can manage one or more daemon processes. Some services run continuously; a oneshot service performs a task and exits.

systemd is the first process started by the system, with PID 1. It manages services and other system resources. Its key features include:

- Starting multiple services in parallel to speed up boot.
- Starting some daemons on demand.
- Managing service dependencies, so a service can wait for a requirement such as networking.
- Tracking related processes using Linux control groups.

A service that is enabled is configured to start automatically at boot. Some services can also start on demand when a socket connection arrives or when a file-system change occurs.

## 2. systemd units

A unit is an object that systemd can manage. Its configuration is stored in a unit file, and its name has a unit type extension.

| Unit type | Extension | Purpose |
| --- | --- | --- |
| Service | `.service` | A service or daemon, such as `sshd.service` |
| Socket | `.socket` | Monitors a communication socket and can start a service when a client connects |
| Path | `.path` | Can activate a service when a specified file-system change occurs |

To see available unit types:

```bash
systemctl -t help
```

You can usually omit `.service` when working with a service:

```bash
systemctl status sshd
```

## 3. List services and unit files

### List loaded, active service units

```bash
systemctl list-units --type=service
```

By default, this shows service units that are loaded and active.

### List all service units, including inactive ones

```bash
systemctl list-units --type=service --all
```

You can filter by a state shown in the `LOAD`, `ACTIVE`, or `SUB` columns:

```bash
systemctl list-units --type=service --state=failed
```

### List installed service unit files

```bash
systemctl list-unit-files --type=service
```

This lists installed service unit files and their boot-related state, including services that are not currently loaded or active.

Common states include:

- `enabled`: configured to start at boot.
- `disabled`: not configured to start at boot.
- `static`: cannot be enabled directly, but another unit can start it.
- `masked`: blocked from starting manually or automatically.

### Understand `list-units` output

| Column | Meaning |
| --- | --- |
| `UNIT` | Unit name |
| `LOAD` | Whether systemd loaded and parsed the unit configuration |
| `ACTIVE` | High-level activation state |
| `SUB` | More detailed, unit-specific state |
| `DESCRIPTION` | Short description of the unit |

`systemctl` without options lists units that are both loaded and active, including unit types other than services.

## 4. Check service status

```bash
systemctl status sshd.service
```

This shows whether the unit is loaded and active, the main process ID, related processes, resource use, and recent log messages.

Useful fields include:

- `Loaded`: whether the unit file has been loaded.
- `Active`: whether and how long the unit has been active.
- `Main PID`: the main process ID.
- `Status`, `Process`, and `CGroup`: additional service and process details.
- `Docs`: related documentation, when available.

Common status words:

| Status | Meaning |
| --- | --- |
| `loaded` | systemd has processed the unit file |
| `active (running)` | The service is running with continuing processes |
| `active (exited)` | A one-time task completed successfully |
| `active (waiting)` | The unit is waiting for an event |
| `inactive` | The service is not running |
| `enabled` | Configured to start at boot |
| `disabled` | Not configured to start at boot |
| `static` | Cannot be enabled directly; another unit may start it |

### Check a specific state

```bash
systemctl is-active sshd
systemctl is-enabled sshd
systemctl is-failed sshd
systemctl --failed --type=service
```

`is-active` checks whether it is currently active. `is-enabled` checks whether it is configured to start at boot. `is-failed` checks for a failed state. The last command lists failed service units.

## 5. Start, stop, restart, and reload services

These commands control a service during the current system session:

```bash
systemctl start UNIT
systemctl stop UNIT
systemctl restart UNIT
systemctl reload UNIT
```

- `start`: starts a service that is stopped.
- `stop`: stops a running service.
- `restart`: stops and then starts the service.
- `reload`: asks a running service to reload its configuration without stopping.

Not every service supports reloading. If you do not know whether it does, use:

```bash
systemctl reload-or-restart UNIT
```

This reloads the configuration if supported; otherwise, it restarts the service.

Remember: starting or stopping a service does not change whether it starts at boot.

## 6. Enable or disable a service at boot

```bash
systemctl enable UNIT
systemctl disable UNIT
```

- `enable`: configures the service to start at boot. It does not start it immediately.
- `disable`: removes that boot-time configuration. It does not stop a service that is already running.

To enable and start a service now:

```bash
systemctl enable --now UNIT
```

To disable it at boot and stop it now:

```bash
systemctl disable --now UNIT
```

Check the boot-time setting with:

```bash
systemctl is-enabled UNIT
```

## 7. Mask and unmask a service

Masking blocks a service from starting, whether manually or automatically. systemd does this by linking the unit name to `/dev/null`.

```bash
systemctl mask UNIT
```

A masked service cannot be started with `systemctl start`. To make it startable again:

```bash
systemctl unmask UNIT
```

Key distinction:

- A disabled service can still be started manually, but is not configured to start at boot.
- A masked service cannot be started manually or automatically.

## 8. Inspect unit dependencies

Some services depend on other units. List the units a service depends on:

```bash
systemctl list-dependencies sshd.service
```

To see units that depend on a particular service, use reverse dependencies:

```bash
systemctl list-dependencies --reverse NetworkManager.service
```

## Exam checklist

When a task asks you to configure or troubleshoot a service:

1. Check its current status:

   ```bash
   systemctl status SERVICE
   ```

2. Start or stop it as required:

   ```bash
   systemctl start SERVICE
   ```

3. Configure boot behaviour separately:

   ```bash
   systemctl enable SERVICE
   ```

4. If the task requires it running now and after reboot, use:

   ```bash
   systemctl enable --now SERVICE
   ```

5. Verify both conditions:

   ```bash
   systemctl is-active SERVICE
   systemctl is-enabled SERVICE
   ```

6. If it fails, inspect `systemctl status SERVICE` and check failed units:

   ```bash
   systemctl --failed --type=service
   ```

Most important distinction: `start` affects the current session; `enable` affects future boots. Use `enable --now` when both are required.
