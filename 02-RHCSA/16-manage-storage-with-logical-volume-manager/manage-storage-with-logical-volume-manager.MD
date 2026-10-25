# Managing Storage with Logical Volume Manager

This chapter teaches you how to build storage from physical devices, create and extend logical volumes, and safely replace or remove LVM components. I’ve skipped the guided exercises, quiz, and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. LVM structure

LVM adds a flexible storage layer between physical disks and file systems:

```text
Physical device → Physical Volume (PV) → Volume Group (VG) → Logical Volume (LV) → File system
```

- **Physical device:** A disk, partition, RAID device, or SAN disk. You can use a whole device or a partition for a PV.
- **Physical Volume (PV):** An initialized device that LVM can use. LVM divides it into physical extents (PEs).
- **Volume Group (VG):** A storage pool made from one or more PVs. A PV can belong to only one VG.
- **Logical Volume (LV):** Storage allocated from the free space in a VG. Applications and file systems use an LV like a disk partition.
- **Logical Extent (LE):** An LV’s storage unit. Normally, each LE maps to one PE. Mirroring is an example of an option that changes this mapping.

LVM makes it possible to extend storage without unmounting a file system or stopping applications in many situations.

### The basic workflow

1. Identify physical devices and initialize them as PVs.
2. Create a VG from one or more PVs.
3. Create LVs from the VG’s free space.
4. Create a file system on an LV and mount it, or initialize it for another purpose such as swap.

## 2. Create an LVM logical volume

### Identify storage

Device names vary. Check the system before selecting a disk or partition:

```bash
lsblk
blkid
cat /proc/partitions
```

### Prepare the physical device

Partitioning is optional: a whole disk can be a PV. If you create a partition, set it to the Linux LVM type. For GPT, the course shows:

```bash
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 1MiB 769MiB
parted /dev/sdb set 1 lvm on
udevadm settle
```

Check the device name and partition layout before making changes; creating a label or partition can make existing data inaccessible.

### Initialize PVs

```bash
pvcreate /dev/sdb1 /dev/sdb2
```

This labels the devices for LVM.

### Create a VG

```bash
vgcreate vg01 /dev/sdb1 /dev/sdb2
```

The first argument is the VG name; the remaining arguments are PVs to include.

### Create an LV

```bash
lvcreate -n lv01 -L 300M vg01
```

- `-n lv01`: name the LV.
- `-L 300M`: request a size. Uppercase `-L` takes a size value.
- `vg01`: VG that supplies the space.

You can instead specify a number of extents with lowercase `-l`:

```bash
lvcreate -n lv01 -l 32 vg01
```

If the VG does not have enough free PEs, LV creation fails. The final LV size rounds up to a PE boundary if the requested size does not match the PE size exactly.

### Create a file system and mount the LV

An LV can be referred to by its traditional device path or its device-mapper path:

```text
/dev/vg01/lv01
/dev/mapper/vg01-lv01
```

Create an XFS file system:

```bash
mkfs -t xfs /dev/vg01/lv01
```

Then create a mount point and mount it:

```bash
mkdir -p /mnt/data
mount /dev/vg01/lv01 /mnt/data
```

To mount it persistently, add an `/etc/fstab` entry, for example:

```text
/dev/vg01/lv01 /mnt/data xfs defaults 0 0
```

Then reload systemd’s configuration and test the mount:

```bash
systemctl daemon-reload
mount /mnt/data
```

You can also use a UUID in `/etc/fstab`. LVM identifies the PVs using UUIDs, so mounting by LV name or UUID works.

## 3. Inspect LVM status

Use the short commands for quick summaries:

```bash
pvs
vgs
lvs
```

Use the detailed commands for more information:

```bash
pvdisplay /dev/sdb1
vgdisplay vg01
lvdisplay /dev/vg01/lv01
```

Key information to check:

- `pvs` / `pvdisplay`: PV, owning VG, PE size, and free or allocated extents.
- `vgs` / `vgdisplay`: total VG size and free space available for new or extended LVs.
- `lvs` / `lvdisplay`: LV path, size, owning VG, and number of LEs.

Before creating or extending an LV, check that the VG has enough free space.

## 4. Extend an LV and its file system

Extending an LV gives it additional capacity from free PEs in its VG. First inspect the VG:

```bash
vgdisplay vg01
```

Increase the LV by 500 MiB:

```bash
lvextend -L +500M /dev/vg01/lv01
```

The `+` means add 500 MiB to the current size. Without `+`, the requested size is the LV’s new total size. Lowercase `-l` specifies a number of extents rather than a size.

Extending the LV alone does not enlarge the file system. Grow both layers.

### Grow XFS

The file system must be mounted. Use its mount point:

```bash
xfs_growfs /mnt/data
```

XFS can grow online but cannot be shrunk.

### Grow ext4

Use the LV device path:

```bash
resize2fs /dev/vg01/lv01
```

`resize2fs` handles ext2, ext3, and ext4. It supports online and offline resizing; ext4 can grow or shrink.

### Grow the LV and file system together

The course notes that `lvextend -r` runs the LV extension and file-system resize together:

```bash
lvextend -r -L +500M /dev/vg01/lv01
```

For XFS, the file system must be mounted for online growth. `fsadm` is another tool that can resize several file-system types.

### Extend swap on an LV

An LV used for swap must be offline to extend it:

```bash
swapoff -v /dev/vg01/swap
lvextend -L +300M /dev/vg01/swap
mkswap /dev/vg01/swap
swapon /dev/vg01/swap
```

`mkswap` creates a new swap signature and UUID. Make sure the persistent `/etc/fstab` entry still points to the correct device or UUID.

Turning off active swap on a live system can cause problems if there is not enough RAM to move its pages. Check swap use first; the course also suggests creating and enabling a replacement swap LV before disabling the old one if needed.

## 5. Add, replace, or remove a PV

### Add capacity to a VG

Prepare and initialize the new device as a PV:

```bash
pvcreate /dev/sdb3
```

Add it to the VG:

```bash
vgextend vg01 /dev/sdb3
```

The VG now has more space available for creating or extending LVs.

### Move data off a PV

To remove a PV from a VG while retaining its data, move its allocated extents to other PVs in the same VG:

```bash
pvmove -A y /dev/sdb1
```

`pvmove` relocates the data from `/dev/sdb1` to free extents elsewhere in that VG. The `-A y` option automatically backs up the VG metadata after the change.

The LVs can remain in use during the move, but the course warns that an unexpected power loss can leave the VG inconsistent. Back up the data before moving extents.

After the PV is empty, remove it from the VG:

```bash
vgreduce vg01 /dev/sdb1
```

If you are decommissioning the device entirely, remove its LVM metadata:

```bash
pvremove /dev/sdb1
```

The usual replacement sequence is:

`pvcreate` new device → `vgextend` VG new PV → `pvmove` old PV → `vgreduce` VG old PV → `pvremove` old PV

## 6. Remove LVM storage

Removing LVM components is destructive. Confirm the data is no longer needed and remove any persistent mount or swap configuration first.

For a file system on an LV:

1. Copy or back up any data that must be kept.
2. Unmount the file system.
3. Remove its `/etc/fstab` entry.
4. Remove the LV.
5. Remove the VG, if it is no longer needed.
6. Remove PV metadata, if the physical devices are being reused.

Example:

```bash
umount /mnt/data
lvremove /dev/vg01/lv01
vgremove vg01
pvremove /dev/sdb1 /dev/sdb2
```

`lvremove` frees the LV’s extents back to its VG. `vgremove` frees the PVs for use in other VGs. `pvremove` deletes the LVM label/metadata from the device.

Do not remove a VG or PV while it still contains LVs or data you need.

## Exam workflow

For a new logical volume:

Identify devices → prepare partitions if needed → `pvcreate` → `vgcreate` → `lvcreate` → `mkfs` → create mount point → mount → configure `/etc/fstab` → verify

For an extension:

Check free VG space → `lvextend` → grow the file system → verify LV and file-system size

Useful verification commands:

```bash
pvs
vgs
lvs
lsblk
df -h /mnt/data
```

Remember: the PV is physical storage initialized for LVM; the VG is the storage pool; the LV is the virtual block device. To make an LV’s new capacity usable, extend both the LV and its file system.
