# Day 1 Linux Fundamentals

## Objective

Build a strong foundation in Linux administration by understanding Linux architecture, the filesystem hierarchy, paths, basic commands, and file/directory operations.

## Environment

* Cloud Platform: Microsoft Azure
* VM: Linux VM
* OS: Ubuntu Server 24.04 LTS
* Shell: Bash
* Access Method: SSH
* Local OS: Windows
* Version Control: Git & GitHub

## Linux Architecture

Basic Linux architecture:

```text
User
  ↓
Shell
  ↓
Linux Utilities / Applications
  ↓
Linux Kernel
  ↓
Hardware
```

### Kernel

The Linux kernel is the core component of the operating system. It manages resources such as:

* CPU
* Memory
* Processes
* Storage
* Networking
* Hardware devices

### Shell

The shell provides an interface for interacting with the operating system through commands.

In this lab, Bash was used.

## Filesystem Hierarchy

| Directory | Purpose                                  |
| --------- | ---------------------------------------- |
| `/`       | Root of the filesystem                   |
| `/home`   | User home directories                    |
| `/root`   | Home directory of the root user          |
| `/etc`    | System and application configuration     |
| `/var`    | Variable data such as logs               |
| `/tmp`    | Temporary files                          |
| `/usr`    | User-space programs and libraries        |
| `/opt`    | Optional/additional application software |
| `/boot`   | Boot-related files                       |

## Commands Practiced

### Identify current user

```bash
whoami
```

### Display hostname

```bash
hostname
```

### Display current directory

```bash
pwd
```

### Check operating system

```bash
cat /etc/os-release
```

### Check kernel version

```bash
uname -r
```

### List files

```bash
ls
ls -l
ls -la
```

### Change directories

```bash
cd /
cd /home
cd ..
cd ~
```

## File and Directory Practice

Created a working directory for the lab and practiced:

* Creating directories
* Creating files
* Writing content to files
* Appending content
* Copying files
* Moving files
* Searching for files
* Viewing file contents

Examples:

```bash
mkdir linux-day1
cd linux-day1

mkdir documents logs scripts backup

touch notes.txt
touch commands.txt

echo "Linux Day 1" > notes.txt
echo "Basic Linux commands" >> notes.txt
```

## Redirection

### `>`

Creates or overwrites the contents of a file.

```bash
echo "Hello Linux" > notes.txt
```

### `>>`

Appends content to an existing file.

```bash
echo "Another line" >> notes.txt
```

## File Search

Practiced using `find` to locate files:

```bash
find /home -name "notes.txt" 2>/dev/null
```

## Key Learnings

* Linux uses a hierarchical filesystem starting from `/`.
* Absolute paths start from `/`.
* Relative paths depend on the current working directory.
* `/etc` contains configuration files.
* `/var` commonly contains changing data such as logs.
* The Linux kernel manages system resources.
* Bash provides a command-line interface to interact with Linux.
* `>` overwrites file content while `>>` appends content.
* `find` can be used to locate files and directories.

## Lab Outcome

Successfully connected to an Azure Linux VM through SSH and practiced fundamental Linux commands, filesystem navigation, directory/file management, and basic file searching.

## Next Steps

Day 2 will focus on:

* Users and groups
* File ownership
* Linux permissions
* `chmod`
* `chown`
* `chgrp`
* ACLs
* Practical access-control scenarios
