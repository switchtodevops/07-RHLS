# Chapter 20: Accessing Network-attached Storage

This chapter teaches you to access directories that an NFS server exports. You can mount them manually, configure them to mount persistently at boot, or use an automounter to mount them when accessed.

## 20.1 Mounting NFS File Systems

### How NFS works

The Network File System (NFS) is a standard network file system used by Linux, UNIX, and similar systems. An NFS server exports directories; an NFS client mounts an exported directory on a local mount point.

RHEL 10 uses NFSv4.2 by default and supports both NFSv3 and NFSv4:

| Version | Key details |
| --- | --- |
| NFSv3 | Uses RPC and can use TCP or UDP. The server runs `rpcbind`, contacted on port 111. |
| NFSv4 | Uses TCP only and does not use the legacy RPC protocol for NFS transactions. |

You can access an NFS export in three ways:

- **Manual mount:** Temporary; does not persist after reboot.
- **`/etc/fstab`:** Persistent; mounts during boot and remains mounted.
- **`autofs`:** Mounts on demand when a user or application accesses the path. This is covered in section 20.3.

An NFS mount configured in `/etc/fstab` can delay boot or cause timeouts when the network or server is unavailable. The `nfs-utils` package is needed for NFS mounting and automounting:

```bash
dnf install nfs-utils
```

RHEL can also mount Windows shares using SMB or CIFS, but their protocol-specific configuration is beyond this chapter.

### Find server exports

For an RPC-based NFSv3 server, query available exports with:

```bash
showmount --exports server
```

NFSv4 uses an export tree. Mounting the tree lets you browse the server’s exported paths, but does not mount every export:

```bash
mount server:/export mountpoint
ls mountpoint
```

You can then mount an individual export by using its full path. A Kerberos-protected export’s path might be visible in the tree, but browsing the tree does not let you mount or access it. Kerberos shares require additional server configuration and credentials.

### Manually mount an export

Create a local mount-point directory if needed:

```bash
mkdir mountpoint
```

`/mnt` is available for temporary mounts; the chapter recommends not using it for long-term or persistent mounts.

Mount the export:

```bash
mount -t nfs -o rw,sync server:/export mountpoint
```

- `-t nfs` specifies the filesystem type. The `server:/export` syntax often lets `mount` infer it.
- `-o` supplies comma-separated options.
- `rw` requests read/write access.
- `sync` makes writes synchronous. The chapter recommends it for production network mounts where each transaction must complete or fail.

Only a privileged user can manually mount or unmount an NFS share. A client’s `rw` option does not grant write access if the server exports the directory as read-only.

### Make a mount persistent with `/etc/fstab`

Add an entry like this to `/etc/fstab`:

```text
server:/export  mountpoint  nfs  rw  0  0
```

Then mount it by referring to the mount point:

```bash
mount mountpoint
```

The `mount` command reads the server and options from `/etc/fstab`. Unlike a manual mount, this configuration persists and is mounted during boot.

### Unmount an export

Unmount using the local mount point:

```bash
umount mountpoint
```

Unmounting does not remove an `/etc/fstab` entry; that entry can mount the share again at boot.

If unmounting reports that the target is busy, a process may have an open file on the share, or a shell may have its current directory inside the mount. Close the application or `cd` out of the mounted directory. If it is still busy, inspect users of the mount:

```bash
lsof mountpoint
```

Close the processes gracefully and retry. In a critical situation, you can kill a process, but this may interrupt its work. The `umount -f` option forces an unmount and can lose unwritten data from open files.

## 20.3 Automounting Storage Devices

### What autofs does

The automounter, provided by the `autofs` service, mounts filesystems and NFS exports on demand. It unmounts them after they are no longer in use.

Unlike `/etc/fstab`, `autofs` does not necessarily mount a filesystem during boot. A user or application’s access to the mount point triggers the mount. The filesystem remains mounted while files are open or processes are using it; after they leave and a short timeout passes, it is unmounted.

Automounting is useful when:

- Users do not have permission to run `mount` themselves.
- Remote filesystems should not delay boot when a network or server is unavailable.
- You want shares to unmount when idle.
- An NFS setup has redundant servers or paths and you want the automounter to select the fastest connection when a new filesystem is requested.

Autofs supports local and remote filesystem types supported by `/etc/fstab`, including NFS and SMB, and their mount options. Automounted filesystems are available to users by default, but access can be restricted with permissions.

### Direct and indirect mounts

| Map type | What the mount point means |
| --- | --- |
| Direct | A known, fixed, absolute path. The mount point is permanent. |
| Indirect | A path created under a base directory when access triggers a mount. The `autofs` service creates and removes the path as needed. |

### Install and configure autofs

Install both packages:

```bash
dnf install autofs nfs-utils
```

The default master map is `/etc/auto.master`. You can change the map-file location in `/etc/autofs.conf`. A common way to add master-map configuration is to place files in `/etc/auto.master.d/`.

A master-map file must use the `.autofs` extension. Its general entry format is:

```text
mountpoint  map-file
```

The entry identifies the base mount directory and the map file containing the mount details. The map file must exist before you start `autofs`.

A map file’s general entry format is:

```text
mountpoint  mount-options  source-location
```

Mount options begin with a dash (`-`) and are comma-separated with no spaces. The automounter creates mount-point directories that do not exist. If directories already exist before the service starts, it does not remove them when it exits. A specified timeout can cause an idle directory to be unmounted after that period.

### Direct map example

In the master map, `/-` indicates direct maps:

```text
/-  /etc/auto.direct
```

For example, `/etc/auto.direct` can contain:

```text
/mnt/docs  -rw,sync  hosta:/shares/docs
```

The direct mount point must be an absolute path. Here, `/mnt` exists independently; `autofs` creates and removes `/mnt/docs` as required.

### Indirect map example

An indirect master-map entry defines a base directory:

```text
/shares  /etc/auto.indirect
```

The map file can contain:

```text
work  -rw,sync  hosta:/shares/work
```

The full mount path is `/shares/work`. Autofs creates and removes the path as needed. The local name does not have to match the server’s directory name.

The server must export the directory with read/write access for the client to have write access. Client mount options cannot grant more access than the server export allows.

### Wildcards in indirect maps

If the server exports several subdirectories that share the same mount options, one wildcard entry can cover them:

```text
*  -rw,sync  hosta:/shares/&
```

The `*` matches the requested indirect-map key; `&` inserts the matched name into the source path. For example, accessing `/shares/work` makes the entry mount `hosta:/shares/work`.

### Mount options to know

- Standard filesystem mount options such as `rw` and `sync` can be used.
- `-fstype=` specifies the filesystem type, such as NFS4 or XFS.
- `-strict` treats mount errors as fatal.

### Start the automounter

After creating the master map and the referenced map files, enable and start the service:

```bash
systemctl enable --now autofs
```

### Alternative: systemd automount

Systemd can create automount units from `/etc/fstab` entries that include `x-systemd.automount`. After editing the file, reload systemd and start the corresponding `.automount` unit:

```bash
systemctl daemon-reload
systemctl start unit.automount
```

The unit name is based on the mount path. For example, `/remote/finance` becomes `remote-finance.automount`. Systemd mounts the filesystem when that directory is first accessed. This method supports only absolute-path mount points, similar to direct maps.

## Exam checklist

- Temporary NFS access: Create a mount point, then use `mount`.
- Persistent mount at boot: Add the server export to `/etc/fstab`.
- On-demand mount and idle unmount: Configure `autofs` or a systemd automount.
- NFSv3 export discovery: Use `showmount --exports server`; remember RPC and `rpcbind` on port 111.
- NFSv4: TCP only; mounting the export tree makes paths browseable but does not mount each export.
- Busy unmount: Check for open files or shells inside the mount; use `lsof mountpoint` to find users.
- Forced unmount: `umount -f` can lose unwritten data.
- Autofs maps: Know the master format (`mountpoint map-file`), map format (`mountpoint options source`), and the difference between direct and indirect maps.
- Wildcards: In an indirect map, `*` matches a requested name and `&` substitutes it into the source path.

Chapter summary: NFS servers export directories for clients to mount. Use `mount` for temporary access, `/etc/fstab` for persistent mounts, and `autofs` or systemd automount for on-demand access. Direct maps use fixed absolute paths; indirect maps create paths under a base directory when accessed.
