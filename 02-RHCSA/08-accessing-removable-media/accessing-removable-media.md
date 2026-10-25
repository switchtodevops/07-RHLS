# Accessing Removable Media

This chapter teaches you to identify a storage device, mount its file system so you can use it, unmount it safely, and search for files. I’ve left out the quiz, guided exercises, and lab as requested. These notes follow your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. Storage device, partition, file system, mount point

Think of it as a sequence:

```text
Disk → partition → file system → mount point → files
```

- A **block device** is the device file Linux uses to access storage. These are usually under `/dev`.
- A **partition** is a section of a disk. It must have a file system before you can mount it.
- A **file system** organizes files and directories on that partition.
- A **mount point** is a directory where Linux makes the file system’s contents available.

Linux puts mounted file systems into one directory tree. It does not assign each one a separate drive letter.

### Common device names

| Device type | Example |
| --- | --- |
| SATA, SAS, USB disk | `/dev/sda`, `/dev/sdb` |
| Partition on that disk | `/dev/sda1`, `/dev/sda2` |
| Virtual disk | `/dev/vda`, `/dev/vdb` |
| NVMe disk and partition | `/dev/nvme0n1`, `/dev/nvme0n1p1` |
| SD or MMC card partition | `/dev/mmcblk0p1` |
| LVM logical volume | `/dev/myvg/mylv` |

Device names can change when disks are added or detected in a different order. For a stable identifier, use the file system’s UUID.

RHEL uses XFS as its default local file system. It also supports ext4; RHEL 10 supports exFAT for removable media.

## 2. Identify devices and check space

Start by inspecting the devices. Don’t assume that a USB drive will always have the same `/dev/sdX` name.

```bash
lsblk
```

Shows disks, partitions, sizes, types, and current mount points.

```bash
lsblk -fp
```

Adds full device paths, file system types, UUIDs, and labels.

```bash
df -h
```

Shows space available on mounted file systems in human-readable units.

```bash
df -h /home
```

Shows which file system contains `/home` and its space usage.

```bash
du -sh /var/log
```

Shows how much space the directory’s contents use. In short: `df` checks a file system; `du` checks files and directories.

## 3. Mount a file system

The mount point directory must exist. `/mnt` is commonly used for temporary mounts.

```bash
mkdir -p /mnt/usb
mount /dev/sdb1 /mnt/usb
```

Check the result:

```bash
lsblk
df -h /mnt/usb
ls /mnt/usb
```

### Mount by UUID

First find the UUID:

```bash
lsblk -fp
```

Then mount using it:

```bash
mount UUID="your-uuid-here" /mnt/usb
```

A UUID is more reliable than `/dev/sdb1` if device detection order changes.

### Important points

- You need root privileges to mount manually.
- The mount point must exist.
- If the mount point already contains files, they are hidden while the file system is mounted there. They become visible again after unmounting.
- A manual mount is temporary. It does not automatically mount the file system again after a reboot. Persistent mounts use `/etc/fstab`, which is covered in later system administration material.
- In a graphical desktop, removable media may mount automatically under `/run/media/USERNAME/LABEL`.

## 4. Unmount safely

Unmount before unplugging removable media so Linux can finish writing cached data.

```bash
umount /mnt/usb
```

You can also unmount by device:

```bash
umount /dev/sdb1
```

The command is `umount`, without an “n”.

If you see `target is busy`, a process is still using the file system. For example, your shell may be inside the mount point. Leave the directory, then try again:

```bash
cd
umount /mnt/usb
```

If it is still busy, check which process is using it:

```bash
lsof /mnt/usb
```

## 5. Locate files

Two commands are useful, but they search differently:

| Command | How it searches | Main limitation |
| --- | --- | --- |
| `locate` | Searches a pre-built file-name database; usually fast | The database may not include recently created files |
| `find` | Searches the live directory tree | Usually slower, but can search by many criteria |

### `locate` examples

```bash
locate passwd
locate -i messages
locate -n 5 passwd
```

`-i` ignores uppercase/lowercase differences. `-n` limits the number of results. If the database is stale, root can update it:

```bash
updatedb
```

### `find` examples

The general pattern is:

```bash
find STARTING_DIRECTORY CRITERIA
```

If you leave out the starting directory, `find` searches from the current directory.

Find by exact name:

```bash
find /etc -name sshd_config
```

Find using a wildcard:

```bash
find / -name '*.txt'
find /etc -name '*pass*'
```

Quote wildcard patterns so the shell does not expand them before `find` receives them.

Find by name, ignoring case:

```bash
find / -iname '*messages*'
```

Find by owner or group:

```bash
find /home -user michael
find /home -group developers
```

Find by file type:

```bash
find /etc -type f    # regular files
find /etc -type d    # directories
find / -type l       # symbolic links
find /dev -type b    # block devices
```

Find by size:

```bash
find /var -size +100M   # larger than 100 MB
find /tmp -size -10k    # smaller than 10 KB
```

Units are case-sensitive: `k` for kilobytes, `M` for megabytes, and `G` for gigabytes. `find -size` rounds sizes to whole units.

Find by modification time:

```bash
find /var/log -mmin -60   # modified less than 60 minutes ago
find /var/log -mmin +60   # modified more than 60 minutes ago
```

Find by permissions:

```bash
find /home -perm 764
find /home -perm -002
```

`-perm 764` matches that exact permission mode. With a leading `-`, the specified permissions must be present, but additional permissions may also be present. For example, `-perm -002` finds files where others have at least write permission.

## Exam task approach

If asked to find files on a removable or mounted file system:

1. Use `lsblk -fp` to identify the correct device, file system, and UUID.
2. Create the requested mount point.
3. Mount the device by its path or UUID.
4. Confirm it is mounted with `lsblk`, `df`, or `findmnt`.
5. Run `find` from the directory named in the task, using the requested criterion.
6. Unmount it before removing the media.

Remember: `lsblk` identifies devices; `df` checks mounted file system space; `du` measures directory usage; `mount` makes files accessible; `umount` safely releases the file system; `find` searches live files; `locate` searches its database.
