# Managing Security with SELinux

This chapter explains how SELinux adds access control to normal Linux permissions, how to manage SELinux modes and file labels, how to adjust policy with Booleans, and how to investigate denials. I’ve skipped the quiz, guided exercises, and lab. These notes follow the teaching sections in your RH199 course. `01-RH199_RHCSA_Rapid_Track_en_10.0.pdf`

## 1. SELinux’s role

Linux file permissions use Discretionary Access Control (DAC): they determine which users or groups can read, write, or execute files.

SELinux adds Mandatory Access Control (MAC). It defines which actions a process may perform on labeled resources. A process can be denied access even when ordinary permissions allow it.

SELinux follows this principle:

An action is denied unless a policy rule explicitly allows it.

That means SELinux can limit damage if a service is compromised. For example, a compromised web server should only access files its policy allows.

SELinux policies are often targeted: they protect specific applications and services. A process with a targeted policy runs in a confined domain. A process without a policy is unconfined and does not receive the same SELinux protection.

## 2. SELinux labels and policy

Resources such as processes, files, directories, and ports have SELinux contexts. A context has four fields:

```text
SELinux-user:role:type:security-level
```

Example:

```text
system_u:object_r:httpd_sys_content_t:s0
```

The targeted policy mainly uses the type field to decide access. Type names commonly end in `_t`.

Examples:

| Resource | Example type | Meaning |
| --- | --- | --- |
| Apache process | `httpd_t` | Web server process domain |
| Web content | `httpd_sys_content_t` | Content the web server may read |
| Apache port | `http_port_t` | Port type allowed for HTTP |
| Temporary file | `tmp_t` | Temporary data type |
| MariaDB process | `mysqld_t` | Database process domain |
| MariaDB data | `mysqld_db_t` | Database data type |

Use `-Z` to display SELinux labels:

```bash
ps axZ
ps -ZC httpd
ls -Z /var/www/html
ls -Zd /var/www/html
```

`ps -Z` shows process labels. `ls -Z` shows file labels; `-d` shows the directory itself.

## 3. SELinux operating modes

| Mode | Behavior |
| --- | --- |
| Enforcing | Enforces policy and blocks denied actions. This is the recommended default. |
| Permissive | Loads policy and logs denials, but does not block the actions. Useful for troubleshooting. |
| Disabled | SELinux is off; actions are neither enforced nor logged, and files are not labeled. Re-enabling it later is difficult. |

### Check and temporarily change the current mode

```bash
getenforce
setenforce 0   # switch to permissive
setenforce 1   # switch to enforcing
```

`setenforce` changes the running mode until reboot; it does not set the persistent default.

RHEL no longer supports setting `SELINUX=disabled` in `/etc/selinux/config`. Disabling SELinux is strongly discouraged.

### Set the default mode

The persistent default is configured in:

```text
/etc/selinux/config
```

Common setting:

```text
SELINUX=enforcing
SELINUXTYPE=targeted
```

The system reads this file at boot. Boot parameters can override it:

- `enforcing=0` or `enforcing=1`: boot permissive or enforcing.
- `selinux=0` or `selinux=1`: disable or enable SELinux at boot.

Red Hat recommends rebooting after changing from permissive to enforcing so services started while permissive are confined on the next boot.

## 4. File contexts and labels

SELinux has default file-context rules. When a file is created:

- If its path matches a policy rule, it receives the label defined by that rule.
- Otherwise, it inherits the label of its parent directory.

A file in an unexpected location can therefore have a label that prevents the intended service from accessing it.

### Copying and moving files

- Copying creates a new inode, so the new file normally receives the destination location’s default label.
- Moving within the same file system usually preserves the original inode and its label.
- `cp -p` preserves file attributes where possible.
- `cp --preserve=context` preserves only the SELinux context.

Always verify the label after copying or moving files:

```bash
ls -Z /path/to/file
```

### Preferred method: add a policy, then apply it

For a permanent custom path, define the intended default label with `semanage fcontext`, then apply it with `restorecon`.

List default file-context rules:

```bash
semanage fcontext -l
```

Add a rule for a directory and its contents:

```bash
semanage fcontext -a -t httpd_sys_content_t '/virtual(/.*)?'
restorecon -Rv /virtual
```

The expression `(/.*)?` matches the directory itself and files below it, including in subdirectories.

To view local file-context customizations:

```bash
semanage fcontext -l -C
```

To remove a custom rule:

```bash
semanage fcontext -d '/virtual(/.*)?'
```

Use `restorecon` to restore labels to the current policy defaults:

```bash
restorecon -v /path/to/file
restorecon -Rv /path/to/directory
```

Useful options include `-v` for reporting changes and `-R` for recursive operation.

### Temporary or testing method: `chcon`

```bash
chcon -t httpd_sys_content_t /virtual
```

`chcon` changes the label on the file directly without changing policy. It can help with testing, but the manual change may be overwritten by `restorecon` or a full system relabel.

Exam rule: For a persistent custom location, use `semanage fcontext` followed by `restorecon`. Use `chcon` mainly for testing or debugging.

The `semanage` and `restorecon` commands are provided by the `policycoreutils` and `policycoreutils-python-utils` packages.

## 5. SELinux Booleans

A Boolean switches an optional part of an SELinux policy on or off. Booleans let you enable a supported application behavior without writing a custom policy.

List all Booleans and their current values:

```bash
getsebool -a
```

Check one Boolean:

```bash
getsebool httpd_enable_homedirs
```

List Booleans with descriptions and current/default values:

```bash
semanage boolean -l
```

Change a Boolean for the current running system:

```bash
setsebool httpd_enable_homedirs on
```

This change is temporary and returns to its persistent setting after reboot.

Make the change persistent:

```bash
setsebool -P httpd_enable_homedirs on
```

Show only locally customized Boolean settings:

```bash
semanage boolean -l -C
```

The course example `httpd_enable_homedirs` allows Apache to serve appropriately labeled home-directory content. Booleans are application-specific; check the service’s SELinux man page for supported options. The `selinux-policy-doc` package provides the SELinux policy documentation man pages.

## 6. Investigate SELinux denials

A denial often means SELinux is correctly blocking an action that policy does not allow. Common causes include:

- A file has the wrong SELinux context, often after being moved or copied.
- A needed Boolean is off.
- The service or application is configured incorrectly.
- The application needs a policy feature that has not been enabled.

SELinux policy does not override normal file permissions or ACLs; check those too.

### Check the file and process labels first

```bash
ps -ZC httpd
ls -Z /var/www/html/file
```

For a known service, check its SELinux man page for the expected process and file-context types.

### Find AVC denials

An SELinux denial is recorded as an AVC (Access Vector Cache) message in:

```text
/var/log/audit/audit.log
```

With `setroubleshoot-server` installed, summaries are also written to `/var/log/messages`. The summary includes a unique ID for the event.

Search recent AVC messages:

```bash
ausearch -m AVC -ts recent
```

Read the detailed report for an event:

```bash
sealert -l UUID
```

Analyze all existing events:

```bash
sealert -a /var/log/audit/audit.log
```

The report identifies the process, target file, attempted action, and possible fixes. Treat its suggestions as guidance: choose the fix that matches the actual cause.

For example, if Apache cannot read a file under `/var/www/html` because it still has a home-directory label, restore the expected label:

```bash
restorecon -v /var/www/html/mypage
```

For a directory tree:

```bash
restorecon -Rv /var/www/html
```

Do not create a new policy simply to silence a denial if the file is in the wrong location or has an incorrect label.

The course shows `audit2allow` and `semodule` as a way to build and install a local policy module for a denial. That is appropriate only when the access is intended and the existing labels, Booleans, and configuration have been checked.

The RHEL web console also has an SELinux page that shows the enforcing state and current SELinux access-control errors. You can open an event to review details and suggested solutions.

## Exam workflow

When a service cannot access a file:

1. Check that SELinux is enabled and note its mode:

   ```bash
   getenforce
   ```

2. Check ordinary permissions and ownership.

3. Inspect the process and file contexts:

   ```bash
   ps -ZC SERVICE
   ls -Z /path/to/file
   ```

4. Check for recent denials:

   ```bash
   ausearch -m AVC -ts recent
   ```

5. Review the specific event:

   ```bash
   sealert -l UUID
   ```

6. If the file is in the correct location but mislabeled, run `restorecon`.
7. If a service needs an optional policy feature, inspect and set the relevant Boolean.
8. For a permanent custom path, add a `semanage fcontext` rule and apply it with `restorecon`.
9. Verify the service works and the labels are correct.

## Quick memory points

- DAC checks user/group permissions; SELinux MAC checks policy labels and allowed actions.
- A denial occurs when there is no policy rule allowing the process-resource-action combination.
- `getenforce` checks the current mode; `setenforce` changes it temporarily.
- `/etc/selinux/config` sets the boot-time default.
- `chcon` changes a label directly; `semanage fcontext` defines a persistent path rule.
- `restorecon` applies the policy’s expected label.
- `setsebool` changes a Boolean now; `setsebool -P` makes the change persistent.
- AVC denials are in `/var/log/audit/audit.log`; use `ausearch` and `sealert` to investigate.
