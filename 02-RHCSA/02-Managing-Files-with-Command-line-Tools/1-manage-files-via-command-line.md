# Managing Files from the Command Line

Since you’ve worked with Linux for years, the commands may feel familiar. For the exam, focus on what each option changes, how links differ, and how Bash expands patterns before a command runs.

## 2.1 Managing files with command-line tools

### Create files and directories

```bash
touch file.txt
```

Creates an empty file if it doesn’t exist. If it already exists, `touch` updates its timestamps.

```bash
mkdir Music Pictures Videos
mkdir -p Thesis/Chapter1 Thesis/Chapter2
```

`mkdir` creates directories. `-p` also creates missing parent directories. Check the path carefully: with `-p`, a typo can create an unintended directory instead of producing an error.

### Copy files and directories

```bash
cp source.txt destination.txt
cp file1.txt file2.txt /backup/
cp -r source_dir /backup/
```

- With multiple source files, the last argument must be a directory.
- `cp` overwrites a same-named destination file by default.
- Directories require recursive copying with `-r`.

### Move or rename files

```bash
mv oldname.txt newname.txt
mv report.txt /backup/
mv -v report.txt /backup/
```

`mv` moves a file; when the source and destination are in the same directory, it renames it. The file’s contents remain unchanged. `-v` shows what the command did.

### Remove files and directories

```bash
rm file.txt
rm -r old_directory
rmdir empty_directory
```

- `rm` removes files.
- `rm -r` removes a directory and its contents.
- `rmdir` removes only an empty directory.
- `rm -i` prompts before each removal.
- `rm -f` forces removal without prompting. If used with `-i`, `-f` takes priority.

Shell deletion has no undo or terminal trash bin. Check your location and target before using recursive removal:

```bash
pwd
ls -l target
```

## 2.2 Guided exercise: Manage files with command-line tools

The exercise practises the basic workflow: create directories, make files with `touch`, move groups of files into the right directories, copy directories with `cp -r`, inspect the result, and clean up.

A useful exam habit is to confirm your source files and destination directory before running `mv` or `cp`. A command with several source files needs an existing directory as its final argument.

## 2.3 Creating links between files

A link gives another name or path for accessing a file.

### Hard link

```bash
ln /home/student/files/target.file /home/student/links/file.hardlink
```

A hard link is another name for the same file data.

Check the inode and link count:

```bash
ls -li /home/student/files/target.file /home/student/links/file.hardlink
```

The paths should show the same inode. The link count increases when the hard link is created. Changes through either name affect the same underlying file. Deleting one name does not remove the data while another hard link remains.

Chapter points to remember:

- Hard links are for regular files.
- They must be on the same filesystem.
- They cannot point to a directory.
- Data is removed only after the last hard link is deleted.

### Symbolic link

```bash
ln -s /tmp /home/student/tempdir
```

A symbolic (soft) link is a separate file that points to a path. It can point to a directory or to a target on another filesystem.

```bash
ls -l /home/student/tempdir
```

The output starts with `l` and shows the target path. If the target is removed, the link remains but becomes dangling. If a file later appears again at that path, the symlink points to it.

To remember the difference:

| Link type | Points to | Same inode as target? | Works across filesystems? |
| --- | --- | --- | --- |
| Hard link | File data | Yes | No |
| Symbolic link | A pathname | No | Yes |

## 2.4 Guided exercise: Create links between files

The exercise has you create both kinds of links, then verify them:

```bash
ln /home/student/files/target.file /home/student/links/file.hardlink
ls -li /home/student/files/target.file /home/student/links/file.hardlink

ln -s /tmp /home/student/tempdir
ls -l /home/student/tempdir
```

For the hard link, compare inode numbers and link counts. For the symbolic link, look for the leading `l` and the `->` target.

## 2.5 Matching file names with shell expansions

Bash processes expansions before running the command. This lets one command act on selected groups of files.

### Pathname expansion (globbing)

| Pattern | Matches |
| --- | --- |
| `*` | Any string, including no characters |
| `?` | Exactly one character |
| `[abc]` | One character from the listed set |
| `[!abc]` or `[^abc]` | One character not in the set |
| `[[:digit:]]` | One digit |
| `[[:upper:]]` | One uppercase letter |
| `[[:lower:]]` | One lowercase letter |
| `[[:alpha:]]` | One letter |

Examples:

```bash
ls a*                 # names beginning with a
ls *a*                # names containing a
ls [ac]*              # names beginning with a or c
ls ????               # names exactly four characters long
ls *[[:digit:]]*      # names containing a digit
```

If a glob matches nothing, Bash passes the pattern along literally. The command may then report that no such file exists.

### Brace expansion

Brace expansion generates strings; it does not check whether matching files exist.

```bash
touch file{1..3}.txt
# file1.txt file2.txt file3.txt

mkdir RHEL{8,9,10}
# RHEL8 RHEL9 RHEL10

echo file{a,b}{1,2}.txt
# filea1.txt filea2.txt fileb1.txt fileb2.txt
```

This is useful for creating related files or directories and for specifying a list of filenames in a command.

### Tilde expansion

```bash
~                 # your home directory
~/Documents       # Documents under your home directory
~manager1         # manager1's home directory
```

### Variable expansion

```bash
USERNAME=operator
echo "$USERNAME"
echo "${USERNAME}_backup"
```

Use braces when adjacent text could be mistaken for part of the variable name. Variable names are case-sensitive.

### Command substitution

`$(command)` runs a command and substitutes its output:

```bash
cp mystery_chapter5.odf "mystery_chapter5_$(date +%F).odf"
```

This names the copy with today’s date. The chapter recommends `$(...)` over older backtick syntax.

### Quoting and escaping

```bash
echo '$HOME'       # prints the literal text $HOME
echo "$HOME"       # prints the value of HOME
echo \$HOME        # escapes the dollar sign
```

- Single quotes stop shell expansions; the text is treated literally.
- Double quotes prevent globbing and word splitting, but allow variable expansion and command substitution.
- A backslash protects the next character from expansion.

## 2.6 Quiz: Match file names with shell expansions

Quiz answers, with the pattern logic:

1. Names ending in `b`: `*b`
2. Names beginning with `b`: `b*`
3. Names whose first character is not `b`: `[!b]*`
4. Names containing `b`: `*b*`
5. Names containing a digit: `*[[:digit:]]*`
6. Names beginning with an uppercase letter: `[[:upper:]]*`
7. Names with at least three characters: `???*`

The key distinction: `?` matches one character, while `*` can match zero or more.

## 2.7 Lab: Manage files from the command line

The lab combines the chapter’s skills in one task. You create and organize files, use brace expansion and globs, move and copy files with relative and home-directory paths, use `cd -` to return to the previous directory, substitute output from `date` into filenames, delete directories with the correct command, and create a hard link.

The best preparation is to practise combining these operations while checking the working directory with `pwd`.

## 2.8 Summary

Be ready to choose the right operation:

- `touch`, `mkdir`, `cp`, `mv`, `rm`, `rmdir`
- `ln` for hard links; `ln -s` for symbolic links
- `*`, `?`, and character classes for filename matching
- `{...}` to generate lists of names
- `~`, variables, and `$(...)` for path and command substitutions

Exam focus: know what the command will affect before you execute it, especially when using wildcards or recursive deletion.
