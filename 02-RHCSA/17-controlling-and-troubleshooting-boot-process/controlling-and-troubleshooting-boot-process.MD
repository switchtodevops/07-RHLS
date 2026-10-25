# Controlling and Troubleshooting the Boot Process

This chapter covers how RHEL boots, how to change GRUB2 settings and select systemd targets, and how to recover from file-system problems that block boot. I’ve skipped the guided exercises and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. Boot process and GRUB2

The boot process starts in firmware and proceeds through the boot loader to the Linux kernel and systemd:

```text
Firmware → GRUB2 → Linux kernel + initramfs → systemd → systemd target → login
```

- **Firmware** is BIOS or UEFI. It loads the boot loader from a disk.
- **GRUB2** is RHEL 10’s default boot loader. It loads the kernel and initramfs, then passes control to the kernel.
- **Kernel** initializes hardware using drivers available in the initramfs.
- **Initramfs** contains the early-boot environment and required files. In RHEL 10, it includes a bootable root file system with a running kernel and systemd unit.
- **systemd** takes over initialization and starts units needed for the selected boot target.

### GRUB2 on UEFI and BIOS

- **UEFI:** Reads boot settings from NVRAM and loads a GRUB2 EFI application. The EFI System Partition is usually mounted at `/boot/efi`.
- **BIOS:** Loads GRUB2’s `boot.img` from a boot sector. That loads `core.img`, which uses additional modules to access the boot disk.

GRUB2’s menu lets you select a kernel entry and edit its kernel arguments. Press `e` to edit an entry. Changes made in this editor apply to one boot only.

## 2. Inspect and change GRUB2 settings

### Inspect boot entries

```bash
grubby --info 1
```

Useful fields in the output:

- `index`: boot-menu entry number, starting from 0.
- `kernel`: kernel file to load.
- `args`: kernel command-line arguments.
- `root`: device containing the kernel and initramfs.
- `initrd`: initial RAM file system image.
- `title`: entry name displayed in the menu.
- `id`: unique identifier for the entry.

### Change the default entry

```bash
grubby --set-default-index 0
```

Verify it by listing boot entries with `grubby --info` or checking the default index:

```bash
grubby --default-index
```

### Add or remove kernel arguments persistently

For example, add `rhgb quiet` to a kernel entry:

```bash
grubby --update-kernel /boot/vmlinuz-KERNEL_VERSION \
  --args="rhgb quiet"
```

Remove those arguments:

```bash
grubby --update-kernel /boot/vmlinuz-KERNEL_VERSION \
  --remove-args="rhgb quiet"
```

The exact kernel path is system-specific. `rhgb quiet` suppresses boot messages for a graphical or quieter boot. Persistent changes made with `grubby` apply to future boots.

## 3. RHEL 10 boot stages

1. GRUB2 loads the kernel and initramfs into memory.
2. GRUB2 passes the kernel arguments and initramfs location to the kernel.
3. The kernel initializes supported hardware and runs `/sbin/init` from the initramfs as PID 1. In RHEL 10, `/sbin/init` links to systemd.
4. The initramfs systemd starts units for `initrd.target`, including mounting the real root file system at `/sysroot`.
5. The kernel switches its root from the initramfs to `/sysroot`.
6. Systemd re-executes using the installed systemd on disk and starts units needed for the default target.

The kernel parameter `init=` can override which init program runs. Use it only when you know the recovery procedure requires it.

## 4. systemd targets

A target groups units needed to reach a system state. Targets can include or depend on other targets.

| Target | Purpose |
| --- | --- |
| `graphical.target` | Multi-user system with graphical and text logins |
| `multi-user.target` | Multi-user system with text logins |
| `rescue.target` | Single-user repair environment with more system services initialized |
| `emergency.target` | Minimal repair shell for when rescue mode cannot start |

For example, `graphical.target` includes `multi-user.target`.

List targets and their status:

```bash
systemctl list-units --type=target --all
```

Show target dependencies:

```bash
systemctl list-dependencies graphical.target
```

### Switch targets while the system is running

```bash
systemctl isolate multi-user.target
```

`isolate` stops units not required by the selected target and starts the units it needs. Only targets with `AllowIsolate=yes` can be isolated.

### Set the default target for future boots

```bash
systemctl get-default
systemctl set-default graphical.target
```

The default is controlled by `/etc/systemd/system/default.target`, usually a symbolic link to `graphical.target` or `multi-user.target`.

### Boot to a different target once

Use this for a one-time recovery boot:

1. Reboot and interrupt the GRUB2 countdown.
2. Select the kernel entry and press `e`.
3. Find the line beginning with `linux`.
4. Append one of these to that line:

   ```text
   systemd.unit=rescue.target
   ```

   or:

   ```text
   systemd.unit=emergency.target
   ```

5. Press `Ctrl+X` to boot with the temporary change.

The change only affects that boot. Both shells require the root password.

Rescue vs. emergency:

- `rescue.target` waits for `sysinit.target` and provides a more initialized environment, including more services and file systems.
- `emergency.target` is more minimal. The root file system is mounted read-only, so remount it read-write before editing files:

  ```bash
  mount -o remount,rw /
  ```

## 5. Shut down or reboot

```bash
systemctl poweroff
systemctl reboot
```

These stop services and unmount file systems before powering off or restarting. `poweroff` and `reboot` are also available as commands.

`halt` stops the system in a safe state but does not power it off.

## 6. File-system problems that stop boot

During boot, systemd reads `/etc/fstab` and mounts the listed file systems. Common problems include:

- A file system is corrupted.
- A device or UUID does not exist or is unavailable.
- The mount-point directory does not exist.
- An `/etc/fstab` entry is mistyped or invalid.

The boot process may time out waiting for a device and enter an emergency shell. It may ask for the root password for maintenance.

### File-system check and repair tools

- **XFS:** `xfs_repair` checks and repairs XFS issues that automatic repair does not fix.
- **ext4:** `fsck.ext4` is a link to `e2fsck`, which can also handle ext2 and ext3. Use `-p` for automatic repair of minor issues:

  ```bash
  fsck.ext4 -p /dev/DEVICE
  ```

There is an `fsck.xfs` binary, but it does not repair XFS; it exists to satisfy boot requirements and exits successfully. Use `xfs_repair` for manual XFS repair.

Run `xfs_repair` or `e2fsck` only on an unmounted file system. If possible, mount and unmount first so the file system can replay its journal. Repairing a mounted file system can cause data loss or corruption.

## 7. Repair `/etc/fstab` from an emergency shell

First find which file systems are mounted:

```bash
mount
```

The root file system may be mounted read-only. Remount it read-write before editing:

```bash
mount -o remount,rw /
```

Then inspect and correct `/etc/fstab`. Check device names and UUIDs with commands such as:

```bash
lsblk
blkid
```

If the problem is a missing mount point, create it:

```bash
mkdir -p /mnt/mountfolder
```

Reload systemd’s view of the configuration, then test all file systems listed in `/etc/fstab`:

```bash
systemctl daemon-reload
mount --all
```

`mount --all` skips file systems already mounted and reports errors. Fix each reported problem before rebooting.

Once it completes without errors, reboot and verify that the system reaches its normal target.

### `nofail` option

The `nofail` mount option allows the system to continue booting if a file system cannot be mounted. The course describes it as a quick test or as an option for a nonessential mount.

Do not use it for a production file system that an application must have: the application might start without its data directory mounted.

## Exam workflow

If boot stops because of a file-system mount:

1. Enter the emergency shell and authenticate as root.
2. Run `mount` to see what is mounted.
3. If `/` is read-only, run:

   ```bash
   mount -o remount,rw /
   ```

4. Check `/etc/fstab`, device names, UUIDs, and mount-point directories.
5. Correct the faulty entry or create the missing directory.
6. Run:

   ```bash
   systemctl daemon-reload
   mount --all
   ```

7. When mounting succeeds, reboot and verify normal boot.

Remember: edit GRUB with `e` and `Ctrl+X` for a one-time boot change; use `grubby` for persistent kernel arguments. `rescue.target` provides a fuller repair environment; `emergency.target` is more minimal and usually has `/` mounted read-only.
