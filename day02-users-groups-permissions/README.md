# Day 2 Linux Users, Groups, Permissions and Access Control

## Objective

Practice Linux user and group administration, file permissions, ownership, umask, SGID, POSIX ACLs, password aging, account expiration, and permission troubleshooting in a Linux environment.

The goal of this lab was to understand not only how to configure permissions, but also how to troubleshoot real-world `Permission denied` issues without using insecure shortcuts such as `chmod 777`.


## Lab Environment

- Cloud Platform: Microsoft Azure
- VM: Linux VM
- Operating System: Ubuntu Linux
- Shell: Bash
- Administration: sudo
- Version Control: Git and GitHub
- Repository: `linux-administration-lab`

**Note:** The current Azure lab VM uses Ubuntu. The Linux administration concepts practiced in this lab are also applicable to RHEL. Package-management commands may differ between Ubuntu and RHEL.


# 1. Inspecting the Current User

Checked the currently logged-in user:

```bash
whoami
```

Checked UID, GID, and group membership:

```bash
id
```

Checked groups:

```bash
groups
```

Checked the account information:

```bash
getent passwd "$(whoami)"
```

Checked the home directory:

```bash
echo $HOME
ls -ld $HOME
```

Checked the current umask:

```bash
umask
```

The Azure/Entra-backed login account had a large UID/GID compared with normal local Linux users.

The current umask was:

```text
0002
```

# 2. Creating Local Linux Users

Created three local users for the lab:

```bash
sudo useradd -m -s /bin/bash linuxadmin
sudo useradd -m -s /bin/bash devuser
sudo useradd -m -s /bin/bash appuser
```

Options used:

```text
-m            Create the user's home directory
-s /bin/bash  Configure Bash as the login shell
```

Verified the users:

```bash
id linuxadmin
id devuser
id appuser
```

Checked `/etc/passwd` information through `getent`:

```bash
getent passwd linuxadmin
getent passwd devuser
getent passwd appuser
```

Verified their home directories:

```bash
ls -ld /home/linuxadmin /home/devuser /home/appuser
```

The local users received normal local UID/GID values, unlike the Azure/Entra-backed login identity.


# 3. Creating and Managing Groups

Created a shared application group:

```bash
sudo groupadd appteam
```

Added all three users as supplementary members:

```bash
sudo usermod -aG appteam linuxadmin
sudo usermod -aG appteam devuser
sudo usermod -aG appteam appuser
```

Verified membership:

```bash
id linuxadmin
id devuser
id appuser
```

Checked the group directly:

```bash
getent group appteam
```

Result:

```text
appteam:x:1005:linuxadmin,devuser,appuser
```

The users retained their individual primary groups while `appteam` became a supplementary group.

This demonstrated why `usermod -aG` is important when adding supplementary group membership.


# 4. Creating a Shared Application Directory

Created an application directory:

```bash
sudo mkdir -p /opt/myapp
```

Changed ownership:

```bash
sudo chown linuxadmin:appteam /opt/myapp
```

Configured permissions:

```bash
sudo chmod 770 /opt/myapp
```

Verified:

```bash
ls -ld /opt/myapp
```

Result:

```text
drwxrwx--- linuxadmin appteam /opt/myapp
```

Permission breakdown:

```text
Owner  → rwx → 7
Group  → rwx → 7
Others → --- → 0
```

Numeric permission:

```text
770
```

For directories, execute (`x`) permission means the user can traverse or enter the directory.


# 5. Numeric and Symbolic Permissions

Created a configuration file as `linuxadmin`:

```bash
sudo -u linuxadmin touch /opt/myapp/app.conf
```

Configured numeric permissions:

```bash
sudo chmod 640 /opt/myapp/app.conf
```

Permission:

```text
rw-r-----
```

Then added group write permission using symbolic mode:

```bash
sudo chmod g+w /opt/myapp/app.conf
```

The resulting permission became:

```text
rw-rw----
```

Numeric equivalent:

```text
660
```

This demonstrated the difference between numeric and symbolic `chmod` operations.


# 6. Troubleshooting Directory Traversal

When the Azure login account attempted to access:

```bash
ls -l /opt/myapp/app.conf
```

it received:

```text
Permission denied
```

The full path was investigated using:

```bash
namei -l /opt/myapp/app.conf
```

The result showed that `/opt/myapp` was:

```text
drwxrwx--- linuxadmin appteam myapp
```

The Azure login account was not a member of `appteam`, so it could not traverse the directory.

This demonstrated that file permissions alone are not enough when troubleshooting access. Permissions on every directory in the path must also be considered.


# 7. SGID on a Shared Directory

The first file created by `linuxadmin` had ownership similar to:

```text
linuxadmin:linuxadmin
```

Even though `/opt/myapp` belonged to the `appteam` group.

To make new files inherit the shared directory's group, SGID was enabled:

```bash
sudo chmod g+s /opt/myapp
```

Verified:

```bash
ls -ld /opt/myapp
```

Result:

```text
drwxrws--- linuxadmin appteam /opt/myapp
```

Numeric equivalent:

```text
2770
```

The leading `2` represents SGID.

Created a new file as `linuxadmin`:

```bash
sudo -u linuxadmin touch /opt/myapp/sgid-test.conf
```

Created another file as `devuser`:

```bash
sudo -u devuser touch /opt/myapp/dev-test.conf
```

The files inherited the `appteam` group:

```text
sgid-test.conf → linuxadmin:appteam
dev-test.conf  → devuser:appteam
```

This demonstrated why SGID is useful for shared team and application directories.


# 8. Umask

Checked the umask for the lab users:

```bash
sudo -u linuxadmin sh -c 'umask'
sudo -u devuser sh -c 'umask'
```

Both returned:

```text
0002
```

With `umask 0002`, typical default permissions are:

```text
Regular file → 664 → rw-rw-r--
Directory    → 775 → rwxrwxr-x
```

A more restrictive temporary umask was tested:

```bash
sudo -u linuxadmin sh -c 'umask 0077; touch /home/linuxadmin/private-file; mkdir /home/linuxadmin/private-dir'
```

Verified:

```bash
sudo -u linuxadmin ls -l /home/linuxadmin/private-file
sudo -u linuxadmin ls -ld /home/linuxadmin/private-dir
```

Results:

```text
private-file → 600 → rw-------
private-dir  → 700 → rwx------
```

This demonstrated how `umask` controls default permissions for newly created files and directories.


# 9. POSIX ACL

The ACL utilities were not initially installed on the Ubuntu VM.

Installed them using:

```bash
sudo apt update
sudo apt install -y acl
```

> On RHEL, the package can be installed using `dnf install acl` when required.

Checked the existing ACL:

```bash
sudo getfacl /opt/myapp/app.conf
```

Initial result:

```text
user::rw-
group::rw-
other::---
```

Provided `appuser` read-only access without changing the file owner or group:

```bash
sudo setfacl -m u:appuser:r-- /opt/myapp/app.conf
```

Verified:

```bash
sudo getfacl /opt/myapp/app.conf
```

Result included:

```text
user::rw-
user:appuser:r--
group::rw-
mask::rw-
other::---
```

Checked using:

```bash
sudo ls -l /opt/myapp/app.conf
```

The file displayed:

```text
-rw-rw----+
```

The `+` indicates an extended ACL exists.


# 10. Testing ACL Permissions

Added content to the configuration file as `linuxadmin`:

```bash
echo "production application configuration" | sudo -u linuxadmin tee /opt/myapp/app.conf
```

Tested read access as `appuser`:

```bash
sudo -u appuser cat /opt/myapp/app.conf
```

The read operation succeeded.

Then tested write access:

```bash
sudo -u appuser sh -c 'echo "modified by appuser" >> /opt/myapp/app.conf'
```

Result:

```text
Permission denied
```

This confirmed that:

```text
user:appuser:r--
```

provided read-only access as intended.


# 11. ACL Mask and Effective Permissions

The ACL mask was deliberately changed to understand effective permissions:

```bash
sudo setfacl -m m:r-- /opt/myapp/app.conf
```

Checked:

```bash
sudo getfacl /opt/myapp/app.conf
```

Result:

```text
group::rw-        #effective:r--
mask::r--
```

Although the group ACL entry contained:

```text
rw-
```

the ACL mask restricted its effective permissions to:

```text
r--
```

This demonstrated an important ACL troubleshooting concept:

```text
Configured ACL permission
        ↓
ACL mask
        ↓
Effective permission
```

For example:

```text
user:someuser:rwx
mask::r-x
```

results in:

```text
Effective permission → r-x
```

The mask was restored:

```bash
sudo setfacl -m m:rw- /opt/myapp/app.conf
```

# 12. Default ACL

Configured a default ACL on `/opt/myapp`:

```bash
sudo setfacl -d -m u:appuser:r-- /opt/myapp
```

Checked the directory ACL:

```bash
sudo getfacl /opt/myapp
```

The output included:

```text
default:user::rwx
default:user:appuser:r--
default:group::rwx
default:mask::rwx
default:other::---
```

Created a new file:

```bash
sudo -u linuxadmin touch /opt/myapp/default-acl-test.conf
```

Checked its ACL:

```bash
sudo getfacl /opt/myapp/default-acl-test.conf
```

The new file automatically inherited:

```text
user:appuser:r--
```

The file also inherited the `appteam` group because SGID was enabled on `/opt/myapp`.

This demonstrated two inheritance mechanisms:

```text
SGID
  ↓
Group ownership inheritance

Default ACL
  ↓
ACL permission inheritance
```

The resulting file showed:

```text
linuxadmin:appteam
```

with an extended ACL.


# 13. Password Status and Aging

Checked the password status of `devuser`:

```bash
sudo passwd -S devuser
```

The account displayed:

```text
L
```

The lab users were created using `useradd` without assigning passwords, so their password authentication was locked.

Checked password-aging information:

```bash
sudo chage -l devuser
```

Configured:

```bash
sudo chage -m 1 -M 90 -W 7 devuser
```

Meaning:

```text
Minimum password age → 1 day
Maximum password age → 90 days
Warning period       → 7 days
```

Verified:

```bash
sudo chage -l devuser
```

The calculated password expiration date was:

```text
Dec 29, 2026
```

# 14. Account Expiration

Checked `appuser`:

```bash
sudo passwd -S appuser
```

Configured temporary account expiration:

```bash
sudo chage -E 2026-11-30 appuser
```

Verified:

```bash
sudo chage -l appuser
```

The account showed:

```text
Account expires → Nov 30, 2026
```

The expiration was then removed:

```bash
sudo chage -E -1 appuser
```

Verified again:

```bash
sudo chage -l appuser
```

Result:

```text
Account expires → never
```

This demonstrated the difference between:

```text
Password locking
Password aging
Password expiration
Account expiration
```

# 15. Production-Style Permission Troubleshooting

## Problem

A developer reported:

> I am getting Permission denied while trying to update `/opt/myapp/app.conf`. I should have access because I am part of the application team.

Affected user:

```text
devuser
```

Instead of immediately changing permissions, the problem was investigated step by step.

### Step 1 — Verify User Identity

```bash
id devuser
```

Result showed that `devuser` belonged to:

```text
appteam
```

So group membership was correct.


### Step 2 — Check the Complete Path

Used:

```bash
sudo -u devuser namei -l /opt/myapp/app.conf
```

The output showed:

```text
/             root:root
/opt          root:root
/opt/myapp    linuxadmin:appteam
app.conf      linuxadmin:linuxadmin
```

`devuser` could traverse `/opt/myapp` because the user belonged to `appteam`.

However, the configuration file belonged to:

```text
linuxadmin:linuxadmin
```

### Step 3 — Inspect the ACL

Checked:

```bash
sudo getfacl /opt/myapp/app.conf
```

The ACL contained:

```text
user::rw-
user:appuser:r--
group::rw-
mask::rw-
other::---
```

There was no ACL entry for `devuser`.

Because `devuser` was:

- Not the file owner
- Not a member of the file's `linuxadmin` group
- Not configured with a named ACL
- Not allowed through `other::---`

the user could not write to the file.


### Step 4 — Apply a Targeted Fix

Instead of using:

```bash
chmod 777 /opt/myapp/app.conf
```

a specific ACL was configured:

```bash
sudo setfacl -m u:devuser:rw- /opt/myapp/app.conf
```

This gave `devuser` only the required read/write permissions.


### Step 5 — Verify as the Affected User

Tested the actual operation as `devuser`:

```bash
sudo -u devuser sh -c 'echo "configuration update by devuser" >> /opt/myapp/app.conf'
```

Verified:

```bash
sudo -u devuser cat /opt/myapp/app.conf
```

Result:

```text
production application configuration
configuration update by devuser
```

The issue was successfully resolved.


# Troubleshooting Approach Learned

For Linux permission problems, I followed this sequence:

```text
Permission denied
      ↓
Check user identity and groups
      ↓
Check directory/path permissions
      ↓
Check file ownership and permissions
      ↓
Check POSIX ACL
      ↓
Check ACL mask/effective permissions
      ↓
Identify the actual cause
      ↓
Apply the minimum required permission
      ↓
Test as the affected user
      ↓
Verify resolution
```

# Important Commands Practiced

```bash
whoami
id
groups
getent
useradd
usermod
groupadd
chmod
chown
umask
namei
getfacl
setfacl
passwd
chage
```

# Key Takeaways

- Linux users are identified internally through UID and GID values.
- Primary and supplementary groups serve different purposes.
- `usermod -aG` can add supplementary group membership without replacing existing memberships.
- `chmod` changes permissions.
- `chown` changes file or directory ownership.
- Directory execute permission controls traversal.
- SGID on a directory helps maintain consistent group ownership in shared directories.
- `umask` controls default permissions for newly created files and directories.
- POSIX ACL provides more granular permissions than traditional owner/group/other permissions.
- The ACL mask can restrict the effective permissions of named users and groups.
- Default ACLs allow new files and directories to inherit ACL entries.
- `chage` manages password aging and account expiration.
- Password locking and account expiration are different concepts.
- `namei -l` is useful for troubleshooting permissions across an entire path.
- Permission problems should be diagnosed before making changes.
- Avoid `chmod 777` as a shortcut for resolving access problems.
- Apply the principle of least privilege.
- Always verify a permission fix by testing as the affected user.


# Day 2 Outcome

Completed hands-on Linux administration covering:

- User administration
- Group administration
- UID and GID
- Primary and supplementary groups
- File and directory permissions
- Numeric and symbolic `chmod`
- Ownership
- Shared application directories
- SGID
- Umask
- POSIX ACL
- ACL masks
- Default ACLs
- Password aging
- Account expiration
- Production-style permission troubleshooting

The lab provided practical experience in identifying and resolving Linux access-control problems while maintaining secure permissions.