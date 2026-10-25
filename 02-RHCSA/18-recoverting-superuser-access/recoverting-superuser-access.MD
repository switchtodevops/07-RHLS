# Recovering Superuser Access

This chapter teaches two ways to regain administrative access when the root password is lost or root access is unavailable. The recommended method uses RHEL rescue media; the alternative boots the installed system directly into a Bash shell. Both methods let you change the root password, then require a full SELinux relabel on the next boot.

## First: decide whether you need recovery mode

If you’re already logged in as root, or as a user with full `sudo` access, you can reset the root password from the running system. When you cannot log in with administrative access, you need to reboot into a recovery environment.

The chapter covers two recovery options:

| Method | What it boots | Chapter’s guidance |
| --- | --- | --- |
| Rescue media | A separate RHEL environment from installation media | Recommended |
| No rescue media | The installed system with `init=/bin/bash` | Risky; use caution |

## Method 1: Use rescue media

You need physical media, such as a USB drive, or an ISO for a virtual machine. Use installation media that matches the installed system’s RHEL version. The chapter identifies the RHEL 10.0 Boot ISO as suitable rescue media.

### Steps

1. Reboot and select the rescue media as the boot device.
2. From GRUB2, select **Troubleshooting → Rescue a Red Hat Enterprise Linux system**.
3. In the rescue menu, choose **Continue** by entering `1`. This mounts the installed system.
4. Change the root directory to the mounted system:

   ```bash
   chroot /mnt/sysroot
   ```

5. Set the new root password:

   ```bash
   passwd root
   ```

   Enter and confirm the new password when prompted.

6. Mark the system for SELinux relabeling:

   ```bash
   touch /.autorelabel
   ```

7. Exit the chroot and rescue environment:

   ```bash
   exit
   exit
   ```

   The first `exit` leaves the chroot; the second leaves rescue mode and reboots. The system relabels files and then reboots again.

### Why the relabel matters

In rescue mode, SELinux is not yet enabled. Some commands—including `passwd`—create a temporary file and replace the original. That replacement file may lack its SELinux context. The `/.autorelabel` file tells the next boot to restore correct SELinux labels across the system, including `/etc/shadow`.

## Method 2: Boot without rescue media

This method edits the GRUB2 kernel command line to start Bash directly. The root filesystem initially mounts read-only, so you must remount it read/write before changing the password. The chapter warns that this method is risky and should be used with caution.

### Steps

1. Reboot and press `Esc` to interrupt the boot-loader countdown.
2. In GRUB2, highlight the kernel entry and press `E` to edit it.
3. Find the kernel command line, which starts with `linux`.
4. Remove any `console=` options from that line.
5. Press `Ctrl+E` to move to the end of the line. Add a space and:

   ```text
   init=/bin/bash
   ```

6. Press `Ctrl+X` to boot with the edited command line.
7. When Bash starts, remount the root filesystem read/write:

   ```bash
   mount -o remount,rw /
   ```

8. Set the new root password:

   ```bash
   passwd
   ```

9. Request an SELinux relabel at the next boot:

   ```bash
   touch /.autorelabel
   ```

10. Continue the boot process:

    ```bash
    exec /sbin/init
    ```

The system completes booting, performs a full SELinux relabel, then reboots again.

## Exam notes: remember the differences

| Detail | Rescue media | No rescue media |
| --- | --- | --- |
| How you enter recovery | Boot the matching RHEL installation media | Edit the GRUB kernel line |
| Recovery environment | Separate rescue environment | Bash started with `init=/bin/bash` |
| Installed system location | `/mnt/sysroot` | `/` |
| Make installed root writable | Rescue menu mounts the system; then `chroot` | `mount -o remount,rw /` |
| Password command | `passwd root` inside the chroot | `passwd` |
| SELinux relabel | `touch /.autorelabel` | `touch /.autorelabel` |
| Resume/reboot | `exit` twice | `exec /sbin/init` |

## Key points to memorize

- Use matching-version rescue media; the chapter recommends this approach.
- In rescue mode, use `chroot /mnt/sysroot` before changing the installed system’s root password.
- With `init=/bin/bash`, the root filesystem is read-only at first. Remount it read/write before running `passwd`.
- Create `/.autorelabel` after either recovery method. The system needs correct SELinux contexts after password recovery.
- Rescue method exit sequence: `exit`, `exit`.
- No-media method resume command: `exec /sbin/init`.
- The GRUB kernel parameter is `init=/bin/bash`; the lesson’s rescue steps also require removing `console=` options from the kernel line.

## Chapter summary

You can regain root access either by booting into rescue mode from matching RHEL media or by starting Bash directly from the GRUB kernel entry. In both cases, make the installed root filesystem writable, reset the password, and create `/.autorelabel` so SELinux contexts are corrected during the next boot.
