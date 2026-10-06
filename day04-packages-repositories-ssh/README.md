# Day 4 — Package Management, Repositories & SSH

## Objective

Learn and practice Linux package management, software repositories, and SSH administration with a focus on production troubleshooting and safe system administration.

## Environment

- Cloud Platform: Microsoft Azure
- VM: Linux-vm01
- Operating System: Ubuntu Server 24.04 LTS
- Package Manager: APT / dpkg
- SSH Server: OpenSSH
- Authentication: Microsoft Entra ID (Azure SSH Login)
- RHEL Comparison: RPM / DNF

## Package Management

The lab VM runs Ubuntu Server 24.04, so APT and dpkg were used for hands-on practice. I also mapped these commands to their RHEL RPM/DNF equivalents.

### Identify the Package Management Tools

```bash
which apt
which dnf
which rpm
apt list --installed 2>/dev/null | head
```

The VM uses the Debian/Ubuntu package management stack. `apt` was available, while `dnf` and `rpm` were not installed.

### Query an Installed Package

I inspected the OpenSSH server package:

```bash
apt show openssh-server
```

I identified which package owns the `sshd` binary:

```bash
dpkg -S /usr/sbin/sshd
```

Result:

```text
openssh-server: /usr/sbin/sshd
```

### Ubuntu and RHEL Command Comparison

| Task | Ubuntu | RHEL |
|---|---|---|
| Query installed package | `dpkg -l` | `rpm -q` |
| Package information | `apt show <package>` | `rpm -qi <package>` |
| List package files | `dpkg -L <package>` | `rpm -ql <package>` |
| Find package owning a file | `dpkg -S <file>` | `rpm -qf <file>` |
| Search package | `apt search <package>` | `dnf search <package>` |
| Install package | `apt install <package>` | `dnf install <package>` |

### Nginx Package Inspection

I verified the Nginx package and binary ownership:

```bash
dpkg -S /usr/sbin/nginx
dpkg -L nginx | head -30
which nginx
ls -l $(which nginx)
apt show nginx 2>/dev/null | head -20
```

The `/usr/sbin/nginx` binary was owned by the `nginx` package.

I also simulated an installation without making any changes:

```bash
apt-get -s install nginx
```

The simulation confirmed that Nginx was already installed and no package changes were required.

## Repositories and Package Troubleshooting

### Repository Configuration

I inspected the repositories configured on the Ubuntu Azure VM:

```bash
grep -rhE '^(URIs:|Suites:|Components:|deb )' /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

The system had the following Ubuntu repositories configured:

- `noble`
- `noble-updates`
- `noble-backports`
- `noble-security`

The enabled repository components included:

- `main`
- `universe`
- `restricted`
- `multiverse`

A Microsoft repository was also configured for Ubuntu Noble.

On RHEL, repository configuration is normally stored under:

```text
/etc/yum.repos.d/
```

A common command to check enabled RHEL repositories is:

```bash
dnf repolist
```

### Package Not Found Troubleshooting

I simulated a package installation failure using a nonexistent package:

```bash
apt-cache policy day4-fake-package
```

Result:

```text
N: Unable to locate package day4-fake-package
```

I then simulated the installation:

```bash
apt-get -s install day4-fake-package
```

Result:

```text
E: Unable to locate package day4-fake-package
```

Instead of immediately assuming that the repositories were broken, I searched the available package metadata:

```bash
apt search day4-fake-package
```

No matching package was found.

I then verified that the repositories were configured correctly and refreshed the package metadata:

```bash
sudo apt update
```

The repository metadata refreshed successfully.

This confirmed that the repositories were reachable and that the failure occurred because `day4-fake-package` does not exist.

### Production Troubleshooting Approach

When a package cannot be found, I would troubleshoot in the following order:

1. Verify that the package name is correct.
2. Search for the package in the available repositories.
3. Check whether the required repositories are configured and enabled.
4. Refresh the repository metadata.
5. Retry the package search or installation.
6. Check repository connectivity or configuration if the problem continues.

### RHEL Equivalent

On RHEL, I would use commands such as:

```bash
dnf repolist
dnf repolist --all
dnf search <package>
dnf list available
sudo dnf clean metadata
sudo dnf makecache
```

I would avoid changing repository configuration until I had confirmed that the repository itself was actually the cause of the problem.

### Key Learning

An `Unable to locate package` or `No match for argument` error does not automatically mean that the repository is broken. The package name, enabled repositories, metadata, and repository connectivity should be verified systematically before making changes.

## SSH Administration and Azure Entra Authentication

### SSH Service Verification

I verified that the OpenSSH server was installed and checked the SSH service:

```bash
sudo systemctl status ssh --no-pager
```

The SSH service was active and running.

The VM uses systemd socket activation, so I also checked:

```bash
sudo systemctl status ssh.socket --no-pager
```

The socket was active and listening on:

```text
0.0.0.0:22
[::]:22
```

The relationship between the units was:

```text
ssh.socket
    ↓
triggers
    ↓
ssh.service
    ↓
sshd
```

This explained why `ssh.service` could show as disabled while SSH was still available through the enabled `ssh.socket`.

### Verify SSH Port

I verified that SSH was listening on TCP port 22:

```bash
sudo ss -lntp | grep ':22'
```

The server was listening on port 22 for both IPv4 and IPv6 connections.

### Validate SSH Configuration

Before making any SSH configuration change, I validated the configuration syntax:

```bash
sudo sshd -t
```

No output indicated that the SSH configuration passed syntax validation.

This is an important production safety check because an invalid SSH configuration could prevent remote access after a reload or restart.

### Inspect Effective SSH Configuration

I checked important effective SSH settings:

```bash
sudo sshd -T | grep -E '^(port|permitrootlogin|passwordauthentication|pubkeyauthentication|authorizedkeysfile) '
```

The effective configuration included:

```text
port 22
permitrootlogin without-password
pubkeyauthentication yes
passwordauthentication no
authorizedkeysfile .ssh/authorized_keys .ssh/authorized_keys2
```

I also verified that the main SSH configuration loads additional configuration files:

```bash
grep -n '^Include' /etc/ssh/sshd_config
```

The configuration included:

```text
/etc/ssh/sshd_config.d/*.conf
```

This demonstrated why checking the effective configuration with `sshd -T` can be more reliable than reading only `/etc/ssh/sshd_config`.

### Azure Entra ID Authentication

The Azure VM uses Microsoft Entra ID integration for SSH authentication.

I checked the effective authentication-related SSH configuration:

```bash
sudo sshd -T | grep -E '^(authorizedkeyscommand|authorizedkeyscommanduser|authenticationmethods|usepam) '
```

The output showed:

```text
usepam yes
authorizedkeyscommand none
authorizedkeyscommanduser none
authenticationmethods any
```

I also verified that the Azure SSH login package was installed and found Azure authentication integration in the PAM configuration.

Relevant PAM entries included:

```text
pam_aad.so
```

This showed that Azure Entra authentication was integrated through the PAM authentication and account-processing stack.

### Verify Successful Entra Login

I inspected SSH logs using:

```bash
sudo journalctl -u ssh -b -n 50 --no-pager
```

The logs showed the successful authentication sequence:

```text
aad_certhandler
pam_aad
Login granted
Accepted publickey
session opened
```

This confirmed that my Microsoft Entra identity was successfully authorized and that an SSH session was created.

### SSH Connectivity Troubleshooting

From Windows PowerShell, I tested TCP connectivity to the Azure VM:

```powershell
Test-NetConnection <VM-PUBLIC-IP> -Port 22
```

The important result was:

```text
TcpTestSucceeded : True
```

This confirmed that TCP port 22 on the Azure VM was reachable from the client.

For security, the actual public IP address is intentionally not stored in this repository.

### Production SSH Troubleshooting Flow

For an SSH timeout, I would troubleshoot in the following order:

1. Verify client-side network connectivity.
2. Verify the VM public IP or DNS information.
3. Check Azure NSG inbound rules for TCP port 22.
4. Check routing and NIC configuration.
5. Check the guest OS firewall.
6. Verify that port 22 is listening.
7. Check `ssh.socket` and `ssh.service`.
8. Validate the SSH configuration.
9. Review SSH logs.
10. Investigate authentication only after confirming that the connection reaches the SSH server.

A timeout normally occurs before authentication, so I would not start by troubleshooting SSH keys or user credentials.

### Safe SSH Configuration Troubleshooting

To practice SSH configuration troubleshooting without risking access to the Azure VM, I created a temporary copy:

```bash
sudo cp /etc/ssh/sshd_config /tmp/day4-sshd_config
```

I deliberately introduced an invalid option into the temporary file:

```bash
echo "InvalidSSHOption yes" | sudo tee -a /tmp/day4-sshd_config
```

I then validated the temporary configuration:

```bash
sudo sshd -t -f /tmp/day4-sshd_config
```

The validation correctly detected:

```text
Bad configuration option: InvalidSSHOption
```

I removed the invalid line:

```bash
sudo sed -i '$d' /tmp/day4-sshd_config
```

Then validated the configuration again:

```bash
sudo sshd -t -f /tmp/day4-sshd_config
```

No output confirmed that the configuration was valid again.

This exercise demonstrated the production-safe workflow:

```text
Edit configuration
        ↓
Validate with sshd -t
        ↓
Fix any errors
        ↓
Validate again
        ↓
Reload only after successful validation
```

The live SSH configuration was never modified or restarted during this exercise.

### SSH Log Analysis

While reviewing SSH logs, I also observed entries such as:

```text
Invalid user
Connection reset
preauth
kex_protocol_error
```

Because the VM has a publicly reachable SSH port, external connection attempts can appear in the logs.

I learned not to treat every error message as the root cause of an incident. During troubleshooting, I should correlate the timestamp, username, source, authentication stage, and reported problem before deciding which log entry is relevant.

### RHEL Comparison

On RHEL, the SSH service is normally managed as:

```bash
sudo systemctl status sshd
sudo systemctl enable --now sshd
sudo sshd -t
sudo journalctl -u sshd
```

The main configuration file is:

```text
/etc/ssh/sshd_config
```

SSH-related authentication and security events may also be available in:

```text
/var/log/secure
```

### Key Learning

SSH troubleshooting should follow a layered approach:

```text
Network
   ↓
Azure NSG / Routing
   ↓
TCP Port 22
   ↓
Firewall
   ↓
SSH Socket / Service
   ↓
SSH Configuration
   ↓
Authentication
   ↓
Logs
```

Following this order helps isolate the actual failure instead of making unnecessary changes to SSH configuration or authentication settings.

## Day 4 Key Takeaways

During this lab, I practiced package management, repository troubleshooting, and SSH administration on an Azure Linux VM.

### Package Management

- Identified the Ubuntu package management tools: APT and dpkg.
- Queried installed packages and package information.
- Identified which package owns a specific file using `dpkg -S`.
- Listed files installed by a package using `dpkg -L`.
- Compared Ubuntu APT/dpkg commands with RHEL RPM/DNF commands.
- Used installation simulation before making package changes.

### Repository Management

- Inspected configured Ubuntu repositories.
- Identified Ubuntu Noble, updates, security, and backports repositories.
- Verified the Microsoft Ubuntu repository.
- Practiced troubleshooting an `Unable to locate package` error.
- Refreshed repository metadata using `apt update`.
- Learned not to assume that a package-not-found error automatically means the repository is broken.

### SSH Administration

- Verified `ssh.service` and `ssh.socket`.
- Confirmed that TCP port 22 was listening.
- Used `sshd -t` to validate SSH configuration safely.
- Used `sshd -T` to inspect the effective SSH configuration.
- Reviewed SSH configuration snippets under `/etc/ssh/sshd_config.d/`.
- Investigated Microsoft Entra ID SSH authentication through PAM.
- Reviewed successful and unsuccessful SSH events using `journalctl`.
- Tested TCP port 22 connectivity from Windows PowerShell.
- Practiced troubleshooting a deliberately broken SSH configuration using a temporary file.

### Production Troubleshooting Lessons

The main troubleshooting principle from this lab was to isolate the problem layer by layer instead of immediately changing configuration.

For SSH:

```text
Client
  ↓
Network
  ↓
Azure NSG / Routing
  ↓
TCP Port 22
  ↓
Guest Firewall
  ↓
SSH Socket / Service
  ↓
SSH Configuration
  ↓
Authentication
  ↓
Logs
```

For package problems:

```text
Verify package name
  ↓
Search available packages
  ↓
Check repositories
  ↓
Refresh metadata
  ↓
Retry
  ↓
Investigate repository connectivity/configuration
```

### Production Safety Practices

- Do not blindly perform full system upgrades on production servers.
- Review available updates and follow the approved change process.
- Validate SSH configuration with `sshd -t` before reload or restart.
- Keep an existing SSH session open while testing SSH configuration changes.
- Avoid restarting remote SSH services unnecessarily.
- Troubleshoot the actual failure before changing permissions, repositories, firewall rules, or SSH settings.
- Correlate logs with timestamps, users, sources, and the reported incident before identifying a root cause.

## Lab Status

**Day 4 — Package Management, Repositories & SSH: Completed**

The Azure VM was stopped and deallocated after completing the lab to avoid unnecessary compute usage.