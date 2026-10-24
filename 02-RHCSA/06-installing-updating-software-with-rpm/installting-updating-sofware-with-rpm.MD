# Installing and Updating Software with RPM

These notes cover the teaching sections and lab, skipping quizzes and guided exercises. The main idea is: RPM packages contain software, RPM queries package information, and DNF manages packages and dependencies from repositories.

## 6.1 Investigating RPM software packages

An RPM package is a software bundle containing files, package information, and sometimes scripts that run during installation, updating, or removal. The RPM database tracks installed packages.

A package filename follows this pattern:

```text
name-version-release.architecture.rpm
```

For example, `x86_64` means 64-bit Intel/AMD architecture; `aarch64` means 64-bit ARM. The version identifies the software version; the release identifies the package build for that version.

Packages are commonly stored in repositories. Red Hat signs its RPM packages with GPG keys so the system can check their integrity and authenticity.

### Query installed packages

`rpm` is a low-level tool. It can inspect installed packages or a downloaded package file, but it does not resolve dependencies from repositories like DNF does.

| Command | What it shows |
| --- | --- |
| `rpm -qa` | All installed packages |
| `rpm -q dnf` | Whether a package is installed and its version |
| `rpm -qi dnf` | Detailed package information |
| `rpm -ql dnf` | Files installed by the package |
| `rpm -qc PACKAGE` | Configuration files |
| `rpm -qd PACKAGE` | Documentation files |
| `rpm -q --scripts PACKAGE` | Install and removal scripts |
| `rpm -q --changelog PACKAGE` | Package change history |
| `rpm -qf /path/to/file` | Which installed package owns the file |

To inspect a downloaded RPM before installing it, use `-p`:

```bash
rpm -qlp package.rpm
```

Here, `-p` means the query is against a package file, rather than the installed package database.

### Install or extract a local RPM

```bash
sudo rpm -ivh package.rpm
```

`-i` installs; `-v` shows details; `-h` displays progress marks. Because RPM does not resolve dependencies from repositories, installation can fail if required packages are missing. DNF is usually better for installation.

To extract files without installing the package:

```bash
rpm2cpio package.rpm | cpio -idv
```

Use `cpio -t` to list the package contents. Be cautious with third-party RPMs: installation scripts can run with root privileges.

## 6.3 Installing and updating packages with DNF

DNF is the recommended package manager for RHEL. It can find, install, update, and remove packages while resolving dependencies from configured repositories.

RHEL 10’s main repositories are:

- **BaseOS:** core operating system packages.
- **AppStream:** applications, languages, and other software streams.

### Find packages

```bash
dnf list http*
dnf search KEYWORD
dnf search all 'web server'
dnf info httpd
dnf provides /var/www/html
```

`dnf search` matches package names and summaries. `dnf search all` also searches descriptions. `dnf provides` finds a package that supplies a file or path.

### Install, update, and remove

```bash
sudo dnf install httpd
sudo dnf update httpd
sudo dnf update
sudo dnf remove httpd
```

DNF shows the planned transaction—including dependencies and removals—before applying it. Review the list, especially before removing a package: DNF may also remove packages that depend on it.

Updating the kernel generally installs a newer kernel alongside the previous one. That lets you boot the older kernel if the new one has a problem.

### Package groups and transaction history

Package groups bundle related software. A group may include mandatory, default, and optional packages.

```bash
dnf group list
dnf group list hidden
dnf group info 'Development Tools'
sudo dnf group install 'Development Tools'
dnf history
```

`dnf history` lists past package transactions.

## 6.5 Enabling DNF software repositories

A repository is a source of packages, such as a Red Hat service, an organization’s server, a website, or a local filesystem.

List repositories and their status:

```bash
dnf repolist all
```

Enable or disable a repository persistently:

```bash
sudo dnf config-manager --enable REPO_ID
sudo dnf config-manager --disable REPO_ID
```

You can also enable or disable a repository for a single DNF command:

```bash
sudo dnf install PACKAGE --enablerepo=REPO_ID
sudo dnf update --disablerepo=REPO_ID
```

### Configure a repository

Repository configuration files usually live in:

```text
/etc/yum.repos.d/
```

For example, a `.repo` file can contain:

```ini
[example]
name=Example Repository
baseurl=https://repo.example.com/rhel/10/x86_64/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-example
```

- The bracketed value is the repository ID.
- `name` is the human-readable name.
- `baseurl` points to the repository.
- `enabled=1` enables it; `enabled=0` disables it.
- `gpgcheck=1` enables signature checking.
- `gpgkey` identifies the trusted key used to verify packages.

You can also add a repository from its URL with `dnf config-manager --add-repo`. RHEL also reads repository sections in `/etc/dnf/dnf.conf`, but the chapter recommends using `.repo` files.

Do not disable GPG checking casually. It removes an important check that the package came from a trusted source and has not been altered. Import keys only from a source you trust.

## 6.7 Lab: Install and update software

The lab combines the chapter skills: configure a custom repository, install software from it, remove an installed package, install a local RPM, and verify the results.

The core workflow is:

1. Create or enable the required repository.
2. Check that DNF can see the package.
3. Install or update it with DNF.
4. Install a local RPM with root privileges if the task specifically requires it.
5. Verify using commands such as `dnf list PACKAGE` or `rpm -q PACKAGE`.

## 6.8 Summary

- RPM is the package format and a low-level query/install tool.
- Use `rpm -q` commands to inspect installed packages and `-p` to query a downloaded RPM.
- Use DNF for repository-based installation, updates, dependency handling, and removal.
- Repository status is checked with `dnf repolist all`; repository definitions are usually `.repo` files under `/etc/yum.repos.d/`.
- GPG verification helps confirm package authenticity. Review DNF’s proposed transaction before accepting it.
