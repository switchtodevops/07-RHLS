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

### Question 2: Create a Repository File

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

### Question 3: Configure SELinux

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

### Question 4: Create Users and Groups

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
$ yum install autofs
$ systemctl enable --now autofs

$ vim /etc/auto.master
/home /etc/auto.misc
$ /home /etc/auto.misc
remoteuser18 -rw,sync serverb.lab.example.com:/rhome/&

$ systemctl restart autofs.service

$ df -hTP /home/remoteuser18
serverb.lab.example.com     9.4G  1.7G  7.7G    18%  /home/remoteuser18
```

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

### Question 7: Configure an NTP Client

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
mkdir /root/find.user
find / -user michael -exec cp -prv {} /root/find.user \;
```

---

### Question 9: Search for a String in a File

Find all occurrences of the string `listen` in `/etc/httpd/conf/httpd.conf` and save the output to `/root/finds`.

**Solution:**

```bash
grep -i listen /etc/httpd/conf/httpd.conf > /root/finds
```

---

### Question 10: Create a User Account with a Specific UID

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
su - john
vim .bashrc
# umask 277

source .bashrc
mkdir dir2
touch file1
ll
```

