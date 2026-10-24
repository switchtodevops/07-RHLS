# Controlling Access to Files

## 5.1 Interpreting Linux file permissions

Each file or directory has an owner, an owning group, and permissions for three classes:

- **User (u):** the file owner
- **Group (g):** members of the owning group
- **Other (o):** everyone else

View permissions and ownership with:

```bash
ls -l file
ls -ld directory    # show the directory itself
```

Example:

```text
-rw-rw-r--. 1 vinod developers 0 Oct 24 18:00 report.txt
```

The first character shows the type (`-` regular file, `d` directory, `l` symbolic link). The next nine characters are permissions in three groups:

```text
-rw- rw- r--
 │   │   └── other
 │   └────── group
 └────────── owner
```

### Read, write, and execute

| Permission | On a file | On a directory |
| --- | --- | --- |
| `r` | Read the contents | List the names inside |
| `w` | Change the contents | Create, delete, or rename entries |
| `x` | Run it as a program | Enter or traverse it |

A directory’s `x` permission is often called search permission. To access a file by its path, you need search permission on its parent directories.

Key rule: Linux chooses the most specific matching class. If you own the file, the owner permissions apply—even if your group permissions are more generous. Otherwise, if you belong to the owning group, group permissions apply; if neither applies, other permissions apply. The classes are not combined.

For example, write access to the directory can allow a user to delete a file even if that user cannot write to the file itself. The sticky bit can restrict who may delete entries.

## 5.3 Managing permissions from the command line

### Change permissions symbolically with `chmod`

```bash
chmod who+permissions file
```

| Symbol | Meaning |
| --- | --- |
| `u`, `g`, `o`, `a` | Owner, group, other, all |
| `+` | Add permission |
| `-` | Remove permission |
| `=` | Set permissions exactly |
| `r`, `w`, `x` | Read, write, execute |

Examples:

```bash
chmod go-rw report.txt     # remove group and other read/write
chmod a+x script.sh        # add execute for everyone
chmod u=rw,g=r,o= report.txt
```

Uppercase `X` adds execute permission only to directories or to files that already have at least one execute bit set. It is useful when changing a directory tree without making every ordinary file executable:

```bash
chmod -R g+rwX project/
```

Use `chmod -R` carefully: it changes permissions throughout the whole tree.

### Change permissions with octal numbers

The values are `r` = 4, `w` = 2, and `x` = 1. Add the values for each class; the digits are in owner, group, other order.

| Mode | Result |
| --- | --- |
| `644` | Owner rw; group and other r |
| `750` | Owner rwx; group r-x; other none |
| `640` | Owner rw; group r; other none |
| `777` | Everyone has rwx |

```bash
chmod 644 report.txt
chmod 750 scripts/
```

### Change ownership

```bash
chown username file
chown :groupname file
chown username:groupname file
chown -R username:groupname directory/
chgrp groupname file
```

Only root can change a file’s user owner. The owner or root can change its group, but a regular user can select only a group they belong to. Use a colon between user and group in `chown`, as in `username:groupname`.

## 5.5 Managing default and special permissions

### `umask`: permissions for new files

New files start with base permissions of `666` (`rw-rw-rw-`); new directories start at `777` (`rwxrwxrwx`). The current `umask` removes permissions from those defaults. It does not add permissions. Regular files do not get execute permission automatically.

For example, `umask 0022` results in:

- New files: `644` (`rw-r--r--`)
- New directories: `755` (`rwxr-xr-x`)

More examples:

| umask | New file | New directory |
| --- | --- | --- |
| `0022` | `644` | `755` |
| `0027` | `640` | `750` |
| `0002` | `664` | `775` |

Check or change the setting for the current shell:

```bash
umask
umask 0027
```

A shell change is temporary. System and user shell configuration can define persistent defaults. In RHEL 9 and later, including this RHEL 10 material, the default for regular users is `0022`.

### Special permissions

Special permissions add behavior beyond the basic `rwx` bits:

| Permission | Octal digit | Symbolic form | Main use |
| --- | --- | --- | --- |
| setuid | `4` | `u+s` | Executable runs with the file owner’s effective user ID |
| setgid | `2` | `g+s` | On directories, new files inherit the directory’s group |
| sticky | `1` | `o+t` | In a shared directory, users are restricted from deleting other users’ files |

Examples:

```bash
chmod u+s program       # set setuid
chmod g+s shared/       # set setgid on a directory
chmod o+t /tmp          # set sticky bit
chmod 2770 shared/      # setgid plus permissions rwxrwx---
chmod 1770 shared/      # sticky plus permissions rwxrwx---
```

In `ls -l`, setuid and setgid appear as `s` where execute normally appears. The sticky bit appears as `t`. Uppercase `S` or `T` means the corresponding execute permission is not set.

The chapter’s examples include `/usr/bin/passwd` with setuid and `/tmp` with the sticky bit.

## 5.7 Lab: Control access to files

The lab combines ownership, permissions, and `umask` to create a shared team directory. The requirements include allowing group members to collaborate, restricting access to nonmembers, and making sure users cannot delete one another’s files.

The scenario has you:

1. Set up a directory with the correct group ownership and access.
2. Use the sticky bit to restrict deletion in that shared directory.
3. Configure files so team members can collaborate.
4. Use ownership and `umask` to give file owners write access, the group read access, and others no access.
5. Verify access as both a team member and a nonmember.

Exam focus: ask whether the permission belongs on the file or its parent directory. File permissions control reading and modifying the file’s contents; directory permissions control listing, entering, and changing entries within that directory.

## 5.8 Summary

- Permissions apply to owner, group, and other, with the most specific matching class taking precedence.
- For directories, `r` lists names, `w` changes directory entries, and `x` permits traversal.
- Use `chmod` with symbolic or octal notation; use `chown` or `chgrp` for ownership.
- `umask` removes permissions from the starting mode for new files and directories.
- Remember the special bits: setuid `4`, setgid `2`, and sticky `1`.
