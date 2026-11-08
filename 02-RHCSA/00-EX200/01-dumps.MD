# RHCSA Mock Exam V10 

## To Be Learned
- DNF repository configuration
- DNF commands
- SELinux

## Exam Format
- Duration: 3 hours
- 3 machines:
  - `servera.lab.example.com`
  - `serverb.lab.example.com`

## Exam UI
Menu (bar on left side of the screen):
- Question paper
- Terminal
- VM

Workflow:
- Open the question paper
- Read the pre-requisites in a different tab

---

## Important Configuration

### Perform on Node 1

### Question 1: Configure Network and Set the Static Hostname

- IP Address: `172.25.250.10`
- Netmask: `255.255.255.0`
- Gateway: `172.25.250.254`
- DNS/Name Server: `172.25.250.254`
- Domain Name: `lab.example.com`
- Hostname: `servera.lab.example.com`

**Solution:**
- The introduction provides the root password
- Take console of node1
- Configure the network using `nmtui`

---

### Question 2: Create a Repository File (Done)

Repositories:
- AppStream: `http://content.example.com/rhel10.0/x86_64/dvd/AppStream`
- BaseOS: `http://content.example.com/rhel10.0/x86_64/dvd/BaseOS`

**Solution:**

```bash
vim /etc/yum.repos.d/local.repo
```

```ini
[BaseOS]
name=baseos
baseurl=http://content.example.com/rhel10.0/x86_64/dvd/BaseOS
enabled=1
gpgcheck=0

[AppStream]
name=appstream
baseurl=http://content.example.com/rhel10.0/x86_64/dvd/AppStream
enabled=1
gpgcheck=0
```

```bash
yum repolist all
yum install httpd* -y
```

---

### Question 3: Configure SELinux (Done)

The webserver is running on non-standard port 82 and has an issue on `servera` with web content. Debug and fix the issue:
- The webserver must be able to serve all existing HTML files located in the `/var/www/html` directory (don't alter or remove any files in this directory)
- The webserver must serve content on port 82
- Make the content accessible through `http://172.25.250.11:82`

**Solution:**

```bash
curl localhost:82
vim /etc/httpd/conf/httpd.conf
# Listen 12.34.56.78:80
Listen 82

firewall-cmd --permanent --add-port=82/tcp
firewall-cmd --reload

semanage port -l | grep http
semanage port -a -t http_port_t -p tcp 82
semanage port -l | grep http

systemctl restart httpd.service
systemctl enable httpd.service
curl localhost:82
```

---

### Question 4: Create Users and Groups (Done)

Create the following:
- A group named `sysmgrs`
- A user `john` who belongs to `sysmgrs` as a supplementary group
- A user `emma` who belongs to `sysmgrs` as a supplementary group
- A user `michael` who does not have access to an interactive shell on the system and who is not a member of `sysmgrs`
- `john`, `emma`, and `michael` should have the password `compedel`

**Solution:**

```bash
groupadd sysmgrs
useradd -G sysmgrs john
useradd -G sysmgrs emma
useradd -s /sbin/nologin michael

passwd --stdin john
passwd --stdin emma
passwd --stdin michael
```

---

### Question 5: Configure Autofs

Configure autofs to automount the home directory of `remoteuser18` as follows:
- `servera.lab.example.com` (`172.25.250.11`) `nfs-exports/rhome` to your system. This contains a pre-configured home directory for the user `remoteuser18`
- `remoteuser18`'s home directory is: `serverb.lab.example.com:/rhome/remoteuser18`
- `remoteuser18`'s home directory should be automounted locally beneath: `/home/remoteuser18`
- The home directory must be writable by the user
- `remoteuser18`'s password is `compedel`

**Solution:**

```bash
yum install autofs
systemctl enable --now autofs

vim /etc/auto.master
# /home /etc/auto.misc

vim /etc/auto.misc
# remoteuser18 -rw,sync serverb.lab.example.com:/rhome/&

systemctl restart autofs.service

df -hTP /home/remoteuser18
# serverb.lab.example.com     9.4G  1.7G  7.7G    18%  /home/remoteuser18
```

---

### Question 6: Configure a Cron Job

User `john` must configure a cron job that runs daily at 12:30 local time and executes `/bin/echo "hello world"`.

**Solution:**

```bash
yum install cronie* -y
systemctl enable --now crond.service

crontab -u john -e
# 30 12 * * *  /bin/echo "hello world"
```

---

### Question 7: Configure an NTP Client (Done)

Configure your system so that it is an NTP client of `classroom.example.com`.

**Solution:**

```bash
vim /etc/chrony.conf
# server classroom.example.com iburst

timedatectl set-ntp true
timedatectl
systemctl restart chronyd.service
chronyc sources

ping classroom.example.com   # should resolve to the IP shown in the chronyc sources output
```

---

### Question 8: Locate Files Owned by a User

Locate all the files owned by `michael` and make a copy of them in the given path `/root/find.user`.

**Solution:**

```bash
  $ mkdir -p /root/find.user
  $ find / -type f -user michael -exec cp -prvf {} /tmp/bkp \;
```

---

### Question 9: Search for a String in a File

Find all occurrences of the string `listen` in `/etc/httpd/conf/httpd.conf` and save the output to `/root/finds`.

**Solution:**

```bash
grep -i listen /etc/httpd/conf/httpd.conf > /root/finds
```

---

### Question 10: Create a User Account with a Specific UID (Done)

Create a user account with UserID `2345`, username `alex`, and password `compedel`.

**Solution:**

```bash
useradd -u 2345 alex
passwd --stdin alex
```

---

### Question 11: Create a Tar Archive

Back up `/usr/local` as `/root/Backup.tar.gz` using gzip compression.

**Solution:**

```bash
tar czvf /root/Backup.tar.gz /usr/local
ls /root/Backup.tar.gz
```

---

### Question 12: Set Default Permissions with umask

- All newly created files for user `john` should default to `-r---------` (read-only, owner only)
- All newly created directories for user `john` should default to `dr-x------` (owner only)

**Solution:**

```bash
  $ su - john
  $ vim .bashrc
    umask 277

  $ source .bashrc
  $ mkdir dir2
  $ touch file1
  $ ll
```
**Logic:**
  - Files: 666 (rw-rw-rw-) — files don’t get execute permission by default.
  - Directories: 777 (rwxrwxrwx) — directories need execute permission to be entered.
  - For example, umask 022 blocks write permission for group and others:
  - File:      666 → 644  (-rw-r--r--)
  - Directory: 777 → 755  (drwxr-xr-x)


### Perform on Node 2

### Question 01: Recover the Root Password (Without Rescue Mode)

Reset the root password on `serverb` without using rescue mode, then allow root to log in over SSH.

**Solution:**

1. Open the `serverb` console and press `Ctrl+Alt+Del` to reboot the system.
2. At the GRUB boot menu, select the main kernel entry and press `e` to edit it.
3. Find the line that starts with `linux`, press `Ctrl+E` to jump to the end of it, and append `init=/bin/bash`.
4. Press `Ctrl+X` (or `F10`) to boot with this change.

```bash
$ mount -o remount,rw /
$ passwd root
$ touch /.autorelabel
$ exec /sbin/init
```

5. After the system reboots and you log back in, allow root login over SSH:

```bash
$ vim /etc/ssh/sshd_config
  PermitRootLogin yes

$ systemctl enable --now sshd
$ systemctl restart sshd
```

---

### Question 02: Create a Repository File (on serverb)

Repositories:
- AppStream: `http://content.example.com/rhel10.0/x86_64/dvd/AppStream`
- BaseOS: `http://content.example.com/rhel10.0/x86_64/dvd/BaseOS`

**Solution:**
```bash
$ ssh root@serverb.lab.example.com
$ vim /etc/yum.repos.d/local.repo
```
  ```ini
  [BaseOS]
  name=baseos
  baseurl=http://content.example.com/rhel10.0/x86_64/dvd/BaseOS
  enabled=1
  gpgcheck=0

  [AppStream]
  name=appstream
  baseurl=http://content.example.com/rhel10.0/x86_64/dvd/AppStream
  enabled=1
  gpgcheck=0
  ```

  ---

### Question 03: Add a Swap Partition

Add an additional swap partition of 756MB to disk `/dev/vdb` on your system. The swap partition should automatically mount when your system boots.

**Solution:**

1. Create a new primary partition with `fdisk`:

```bash
fdisk /dev/vdb
# n (new partition), p (primary), accept the default start, size +756M
# w (write changes)
```

2. Initialize the swap area:

```bash
mkswap /dev/vdb1
```

3. Add the swap entry to `/etc/fstab`:

```bash
vim /etc/fstab
# /dev/vdb1   swap  swap  defaults  0 0
```

4. Enable swap and list it:

```bash
swapon -a
swapon -s
```

---

### Question 04: Resize a Logical Volume

Resize the logical volume `v0` and its file system to 230MiB, making sure the filesystem contents remain intact.
Note: Partitions are seldom exactly the requested size, so a size within the range 217MiB to 243MiB is acceptable.

**Solution:**

```bash
  $ lvs
  $ vgs
  $ lvresize -L 230M /dev/vogroup/v0
  $ resize2fs /dev/vogroup/v0     # for ext2/3/4 file system 
  $ xfs_growfs /mountpoint        # for xfs file system
```

---

### Question 05: Configure Flatpak Repositories

Configure Flatpak on your system so that user `student` can install Codium from the Flathub repository. The following requirements must be met:
- A Flatpak repository has been made available from: `https://flathub.org/repo/flathub.flatpakrepo`
- The Flatpak repository should be named `flatdb`
- The repository must be configured only for the user `student`
- Install the `codium` application for user `student`

**Solution:**

```bash
  $ su - student
  $ flatpak remote-add --user --if-not-exists flatdb https://flathub.org/repo/flathub.flatpakrepo
  $ flatpak install --user flatdb codium
  # Which do you want to use (0 to abort) [0-2]: 2
```
---
### Question 06: Create a Logical Volume

Create a new logical volume according to the following requirements:
- The logical volume is named `qa` and belongs to the `qagroup` volume group and has a size of 60 extents. 
- Logical volume in the `qagroup` volume group should have an extent size of 16MiB
- The physical volume for the logical volume must be placed on disk `/dev/vdb`
- Format the new logical volume with an ext3 filesystem and the logical volume should be automatically mount under /mnt/qa system boot time. 

**Solution:**

```bash
  $ fdisk /dev/vdb
    n (new partition), p (primary)
    t -> select the new partition -> 8e (Linux LVM)
    w (write changes)

  $ partprobe /dev/vdb
  $ pvcreate /dev/vdb2
  $ vgcreate -s 16M qagroup /dev/vdb2
  $ lvcreate -n qa -l 60 qagroup
  $ lvs

  $ mkfs.ext3 /dev/qagroup/qa
  $ mkdir /mnt/qa

  $ vim /etc/fstab
    /dev/mapper/qagroup-qa    /mnt/qa   ext3  defaults  0 0

  $ systemctl daemon-reload
  $ mount -a
  $ lsblk
```

### Question 07: Configure system tunning
Choose the recommended tuned profile for your system and set it as default

```bash
  $ systemctl status tuned 
  $ tuned-adm recommend 
  $ tuned-adm profile virtual-guest 
  $ tuned-adm active 
```