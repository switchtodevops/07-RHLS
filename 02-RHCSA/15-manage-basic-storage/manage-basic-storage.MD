# Managing Basic Storage

This chapter covers creating disk partitions, formatting and mounting file systems, and creating swap space. I’ve skipped the quiz, guided exercises, and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. Disk partitions

Partitioning divides a disk into separate logical areas. It can help separate operating system files, user data, and swap space, or limit how much storage a particular use can consume.

Before creating or changing partitions, identify the correct disk:

```bash
lsblk
parted /dev/sdb print
```

`parted` changes the partition table of the disk you specify. Use the correct device: creating a new disk label or partition layout can make existing data inaccessible.

### MBR and GPT

| Scheme | Main points |
| --- | --- |
| MBR (`msdos`) | Standard on legacy BIOS systems. Supports four primary partitions, or three primary plus one extended partition containing logical partitions. Has a 2 TiB disk/partition size limit. |
| GPT (`gpt`) | Standard on UEFI systems. Supports up to 128 partitions and disks/partitions up to 8 ZiB. Stores a backup partition table and uses checksums to detect errors. |

### Inspect a partition table

```bash
parted /dev/sdb print
parted /dev/sdb unit s print
```

`parted` shows sizes in decimal units by default. The `unit` option can use sectors (`s`), bytes (`B`), or binary/decimal units such as `MiB`, `GiB`, `MB`, and `GB`.

You can use `parted` interactively:

```bash
parted /dev/sdb
```

Then use `print`, `help`, and `quit` at the `(parted)` prompt.

### Create a disk label

For a new MBR partition table:

```bash
parted /dev/sdb mklabel msdos
```

For GPT:

```bash
parted /dev/sdb mklabel gpt
```

`mklabel` replaces the existing partition table. Do this only when you intend to reuse the disk without its existing layout.

### Create a partition

For GPT, an example interactive sequence is:

```text
(parted) mkpart
Partition name? userdata
File system type? xfs
Start? 2048s
End? 1000MB
```

For MBR, `mkpart` also asks for the partition type, such as `primary`, `extended`, or `logical`.

The file-system type entered in `parted` is a partition type label. It does not create or format the file system.

A noninteractive example:

```bash
parted /dev/sdb mkpart userdata xfs 2048s 1000MB
```

For MBR, the arguments differ because you specify the partition type:

```bash
parted /dev/sdb mkpart primary xfs 2048s 1000MB
```

Partition starts should be aligned to disk boundaries; a start sector that is a multiple of 2048 is commonly suitable. `parted` rounds values to meet disk constraints and warns about misalignment.

After making a partition, wait for the kernel to detect it and create the device file:

```bash
udevadm settle
lsblk
```

### Delete a partition

Check the partition number first:

```bash
parted /dev/sdb print
```

Then delete it:

```bash
parted /dev/sdb rm 1
```

This removes the partition from the partition table. If it held a file system, its data may become inaccessible.

## 2. Create a file system

After creating a partition, format it with a file system. RHEL recommends XFS as the default; ext4 is also supported.

```bash
mkfs.xfs /dev/sdb1
```

For ext4:

```bash
mkfs.ext4 /dev/sdb1
```

Formatting creates a new file system and can destroy existing data on that partition. Check the target device before running `mkfs`.

## 3. Mount a file system

A file system must be mounted into the directory tree before users and programs can access its files.

Create a mount point and mount the partition:

```bash
mkdir -p /mnt/data
mount /dev/sdb1 /mnt/data
```

Check the result:

```bash
mount | grep sdb1
lsblk
```

A manual mount lasts only until the file system is unmounted or the system reboots.

## 4. Mount automatically with `/etc/fstab`

To mount a file system at boot, add an entry to `/etc/fstab`. The file has six whitespace-separated fields:

```text
device  mount-point  filesystem-type  options  dump  fsck-order
```

Example:

```text
UUID=your-uuid  /mnt/data  xfs  defaults  0 0
```

| Field | Purpose |
| --- | --- |
| 1 | Device, preferably specified by its UUID |
| 2 | Mount point; the directory must exist |
| 3 | File-system type, such as `xfs` or `ext4` |
| 4 | Comma-separated mount options; `defaults` is a common choice |
| 5 | `dump` backup flag |
| 6 | File-system check order |

For the last field:

- XFS: use `0`; XFS does not use `fsck` to check its file-system status.
- ext4 root file system: use `1`.
- Other ext4 file systems: use `2`.

### Find the UUID

```bash
lsblk --fs
```

UUIDs are stored in the file system and are more reliable for `/etc/fstab` than `/dev/sdb1`, because device names can change if disks are detected in a different order.

### Validate the entry before rebooting

An incorrect `/etc/fstab` entry can cause boot problems. Test it first:

```bash
umount /mnt/data
mount /mnt/data
```

When you specify only the mount point, `mount` reads its settings from `/etc/fstab`.

You can also check the file for errors:

```bash
findmnt --verify
```

After editing `/etc/fstab`, reload systemd’s configuration:

```bash
systemctl daemon-reload
```

## 5. Swap space

Swap is disk space used by the kernel’s memory-management system to hold inactive memory pages. Virtual memory is the combination of RAM and swap.

When RAM use exceeds a limit, the kernel can move inactive pages to swap and reuse that RAM for other processes. Reading those pages back from disk is slower than using RAM, so swap supplements memory but is not a replacement for adequate RAM.

### Swap size guidance from the course

| Physical RAM | Swap guidance | If allowing for hibernation |
| --- | --- | --- |
| 2 GB or less | Twice RAM | Three times RAM |
| Between 2 GB and 8 GB | Equal to RAM | Twice RAM |
| Between 8 GB and 64 GB | At least 4 GB | 1.5 times RAM |
| More than 64 GB | At least 4 GB | Hibernation not recommended |

These are course guidelines; workload and application requirements also affect sizing. Hibernation needs swap large enough to save the RAM contents.

### Create a swap partition

Create a partition with `parted` and set its type label to `linux-swap`. For GPT, for example:

```text
(parted) mkpart
Partition name? swap1
File system type? linux-swap
Start? 1001MB
End? 1257MB
```

The type label helps identify the partition’s purpose. You must still initialize it as swap.

Wait for the partition device to appear:

```bash
udevadm settle
```

### Format and activate swap

Initialize the swap area:

```bash
mkswap /dev/sdb2
```

`mkswap` writes a swap signature to the device; it does not format it with a regular file system.

Activate it now:

```bash
swapon /dev/sdb2
```

Check active swap:

```bash
swapon --show
free
```

Deactivate it:

```bash
swapoff /dev/sdb2
```

If the kernel cannot move the pages using that swap area elsewhere, `swapoff` fails and the swap remains active.

### Make swap persistent

Add an entry to `/etc/fstab`:

```text
UUID=your-swap-uuid  swap  swap  defaults  0 0
```

The first `swap` after the UUID is the placeholder in the mount-point field; swap is not mounted into a directory. The next `swap` is its type. `defaults` includes automatic activation at boot. The final two fields are zero because swap does not need a file-system backup or check.

Get the UUID from the output of `mkswap` or `lsblk --fs`. Reload systemd after editing `/etc/fstab`:

```bash
systemctl daemon-reload
swapon -a
swapon --show
```

### Swap priority

The kernel uses the highest-priority swap area first. The default priority is `-2`. If multiple swap areas have the same priority, the kernel uses them in round-robin fashion.

Set a priority in `/etc/fstab` with the `pri` option:

```text
UUID=swap-uuid  swap  swap  pri=10  0 0
```

Check priorities with:

```bash
swapon --show
```

## Exam task workflow

For a new file system:

1. Identify the disk and existing layout:

   ```bash
   lsblk
   parted /dev/sdb print
   ```

2. Create the partition with `parted`.
3. Wait for the device file:

   ```bash
   udevadm settle
   ```

4. Create a file system:

   ```bash
   mkfs.xfs /dev/sdb1
   ```

5. Create the mount point and test a manual mount:

   ```bash
   mkdir -p /mnt/data
   mount /dev/sdb1 /mnt/data
   ```

6. For persistence, add the UUID-based `/etc/fstab` entry.
7. Validate the entry before rebooting:

   ```bash
   findmnt --verify
   umount /mnt/data
   mount /mnt/data
   ```

For swap, the key sequence is:

Create partition → `mkswap` → `swapon` → `/etc/fstab` entry → verify with `swapon --show`

Key distinction: `parted mkpart` creates a partition; `mkfs.xfs` or `mkfs.ext4` creates a regular file system; `mkswap` initializes swap space.
