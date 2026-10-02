# Day 3 - Linux Processes, Services and systemd

## Objective

Practice Linux process management, system performance monitoring, service management with systemd, and production-style service troubleshooting.

The lab focuses on identifying and managing processes, monitoring CPU, memory and disk activity, working with process priorities, managing systemd services and targets, analyzing service logs, and troubleshooting service failures.

## Environment

- Cloud Platform: Microsoft Azure
- VM: Linux-vm01
- Operating System: Ubuntu Server
- Shell: Bash
- Service Manager: systemd
- Remote Access: SSH

## Process Management

### Checking PID and PPID

Checked the current Bash shell process:

```bash
ps -p $$ -o user,pid,ppid,stat,cmd
```

Observed:

- Bash PID: `1235`
- Parent PID: `1234`
- Process state: `S` (interruptible sleep)

Checked the parent process and process hierarchy:

```bash
ps -p 1234 -o user,pid,ppid,stat,cmd
pstree -ps $$
```

The process tree showed:

```text
systemd(1)
└── sshd
    └── sshd
        └── sshd
            └── bash
                └── pstree
```

This demonstrated how the SSH service created the login session and Bash shell.

### Background Process and Job Control

Started a background process:

```bash
sleep 300 &
jobs -l
```

Verified the process:

```bash
ps -ef | grep '[s]leep 300'
```

This demonstrated the difference between a shell job ID and a Linux process ID (PID).

### Process States and Signals

Stopped the test process using `SIGSTOP`:

```bash
kill -STOP <PID>
```

The process entered the `T` (stopped) state.

Resumed it using `SIGCONT`:

```bash
kill -CONT <PID>
```

After resuming, the `sleep` process returned to the `S` (interruptible sleep) state.

Terminated the process gracefully using `SIGTERM`:

```bash
kill -15 <PID>
```

Verified that the process no longer existed:

```bash
ps -p <PID> -o pid,ppid,stat,cmd
```

### Key Learning

- PID uniquely identifies a running process.
- PPID identifies the process that created the child process.
- `S` represents interruptible sleep.
- `T` represents a stopped process.
- `SIGSTOP` suspends a process.
- `SIGCONT` resumes a stopped process.
- `SIGTERM` should normally be attempted before `SIGKILL` in production.

## System Performance Monitoring

### CPU and Memory Usage with ps

Checked processes consuming the most CPU:

```bash
ps aux --sort=-%cpu | head
```

Checked processes consuming the most memory:

```bash
ps aux --sort=-%mem | head
```

The Azure VM was lightly loaded, with no process showing significant CPU consumption.

### Real-Time Monitoring with top

Used `top` to review system load, process states, CPU utilization, memory, and swap.

```bash
top
```

Observed during the lab:

- Load average was approximately `0.12, 0.11, 0.09`.
- CPU was almost `100%` idle.
- I/O wait was `0%`.
- No zombie processes were present.
- Approximately `645 MiB` of memory was available.
- No swap was configured.

### Memory Monitoring

Checked memory utilization:

```bash
free -h
```

Observed approximately:

```text
Total memory:     887 MiB
Used memory:      241 MiB
Free memory:      290 MiB
Available memory: 645 MiB
Swap:             0 B
```

The `available` value was more useful than looking only at completely free memory because Linux can reclaim some cached memory when applications require it.

### CPU, Process Queue and I/O Monitoring

Used `vmstat` to collect five samples at two-second intervals:

```bash
vmstat 2 5
```

Important columns reviewed:

- `r` - runnable processes
- `b` - blocked processes
- `si` / `so` - swap activity
- `bi` / `bo` - block device I/O
- `us` - user CPU
- `sy` - system CPU
- `id` - idle CPU
- `wa` - I/O wait

The samples showed no CPU, memory, swap, or I/O pressure.

### Disk I/O Monitoring

Checked extended disk statistics:

```bash
iostat -xz 2 3
```

Important metrics reviewed:

- `r/s` and `w/s` - reads and writes per second
- `r_await` and `w_await` - read and write latency
- `aqu-sz` - average I/O queue size
- `%util` - device utilization

The Azure disk showed low utilization, low latency, and no significant I/O queue during the test.

### Troubleshooting Approach

For a server with high load average, I would not immediately assume that CPU is the root cause.

My investigation would follow this approach:

```text
uptime
   |
   v
top
   |
   +--> High CPU -> Identify CPU-intensive processes
   |
   +--> Memory pressure -> Check free, available memory and swap activity
   |
   +--> High I/O wait / blocked processes -> Check vmstat and iostat
```

This helps identify whether the underlying problem is CPU contention, memory pressure, or storage I/O before taking corrective action.

## Process Priority with nice and renice

Linux nice values influence CPU scheduling priority for normal processes.

The typical nice range is:

```text
-20 = higher scheduling priority
  0 = default
 19 = lower scheduling priority
```

### Starting a Process with a Nice Value

Started a test process with a nice value of `10`:

```bash
nice -n 10 sleep 300 &
```

Verified the priority:

```bash
ps -p <PID> -o pid,ppid,ni,stat,cmd
```

The process showed:

```text
NI = 10
```

### Changing the Priority of a Running Process

Changed the nice value of the running process from `10` to `15`:

```bash
renice 15 -p <PID>
```

Verified the new value:

```bash
ps -p <PID> -o pid,ppid,ni,stat,cmd
```

The process showed:

```text
NI = 15
```

Increasing the nice value lowered the process's CPU scheduling priority while allowing it to continue running.

### Key Learning

- `nice` starts a process with a specified nice value.
- `renice` changes the nice value of an existing process.
- A lower nice value represents higher CPU scheduling priority.
- A higher nice value represents lower CPU scheduling priority.
- Process priority should be adjusted only after understanding the workload and the actual resource bottleneck.

## systemd Service Management

### Verifying systemd

Verified that systemd was running as PID 1:

```bash
ps -p 1 -o pid,ppid,comm,args
```

Observed:

```text
PID  PPID  COMMAND
1    0     systemd
```

Checked the overall systemd state:

```bash
systemctl is-system-running
```

Result:

```text
running
```

### Inspecting the SSH Service

Checked the SSH service without stopping or restarting it because it was being used for remote access to the Azure VM:

```bash
systemctl status ssh --no-pager
systemctl is-active ssh
systemctl is-enabled ssh
```

The SSH service was:

```text
Active: active (running)
Enabled: disabled
```

Further investigation showed that SSH was being activated through `ssh.socket`.

```bash
systemctl status ssh.socket --no-pager
systemctl is-enabled ssh.socket
```

The socket was:

```text
Active: active (running)
Enabled: enabled
Listen: TCP port 22
Triggers: ssh.service
```

This demonstrated that a service can be active even when the service unit itself is disabled because another systemd unit, such as a socket, can activate it.

### systemd Targets

Checked the configured default target:

```bash
systemctl get-default
```

Result:

```text
graphical.target
```

Listed currently active targets:

```bash
systemctl list-units --type=target --state=active --no-pager
```

Both `graphical.target` and `multi-user.target` were active along with other system targets.

A server intended to boot permanently into a non-GUI multi-user environment can be configured with:

```bash
sudo systemctl set-default multi-user.target
```

This command was reviewed but not applied to the Azure lab VM.

### Service Logs with journalctl

Reviewed SSH service logs for the current boot:

```bash
sudo journalctl -u ssh -b --no-pager | tail -10
```

The logs showed successful Azure/Entra-backed SSH authentication, including successful public-key authentication and session creation.

The exercise demonstrated that individual log messages should be interpreted in context rather than treating every warning or unusual message as the root cause of an incident.

## Custom systemd Service

### Creating a Test Service

Created an executable Bash script:

```bash
sudo nano /usr/local/bin/day3-demo.sh
```

Script:

```bash
#!/bin/bash

while true
do
    echo "Day 3 demo service is running"
    sleep 30
done
```

Made the script executable:

```bash
sudo chmod +x /usr/local/bin/day3-demo.sh
```

Created a custom systemd unit:

```bash
sudo nano /etc/systemd/system/day3-demo.service
```

Unit configuration:

```ini
[Unit]
Description=Day 3 systemd demo service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/day3-demo.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Reloaded systemd after creating the unit:

```bash
sudo systemctl daemon-reload
```

Started and verified the service:

```bash
sudo systemctl start day3-demo
systemctl status day3-demo --no-pager
systemctl is-active day3-demo
```

The service successfully entered:

```text
Active: active (running)
```

### Viewing Custom Service Logs

Checked the service logs:

```bash
sudo journalctl -u day3-demo -n 10 --no-pager
```

The journal showed messages from the script every 30 seconds:

```text
Day 3 demo service is running
```

This demonstrated how standard output from a systemd-managed service can be captured by the system journal.

### Enabling the Service

Enabled the service for automatic startup:

```bash
sudo systemctl enable day3-demo
```

Verified:

```bash
systemctl is-enabled day3-demo
```

Result:

```text
enabled
```

Enabling the service created a symbolic link under:

```text
/etc/systemd/system/multi-user.target.wants/
```

This associated the service with `multi-user.target`.

## Production Troubleshooting Scenario

Simulated a service failure by changing the valid `ExecStart` path:

```ini
ExecStart=/usr/local/bin/day3-demo.sh
```

to an invalid path:

```ini
ExecStart=/usr/local/bin/day3-missing.sh
```

Reloaded the systemd configuration:

```bash
sudo systemctl daemon-reload
```

Attempted to start the service and then investigated its state:

```bash
sudo systemctl start day3-demo
systemctl status day3-demo --no-pager
```

The service showed:

```text
Active: failed (Result: exit-code)
status=203/EXEC
```

### Investigating the Failure

Checked recent service logs:

```bash
sudo journalctl -u day3-demo -n 10 --no-pager
```

The journal showed repeated execution failures and restart attempts:

```text
Main process exited, code=exited, status=203/EXEC
Failed with result 'exit-code'
Scheduled restart job
Start request repeated too quickly
```

Because the unit contained:

```ini
Restart=on-failure
```

systemd attempted to restart the failed service several times before stopping further rapid restart attempts.

Verified the executable paths:

```bash
ls -l /usr/local/bin/day3-demo.sh
ls -l /usr/local/bin/day3-missing.sh
```

The valid script existed and was executable, while the configured `day3-missing.sh` path did not exist.

### Root Cause

The `ExecStart` directive pointed to a nonexistent executable:

```text
/usr/local/bin/day3-missing.sh
```

This caused systemd to fail to execute the service command and report `203/EXEC`.

### Resolution

Corrected the unit file:

```ini
ExecStart=/usr/local/bin/day3-demo.sh
```

Reloaded the systemd configuration:

```bash
sudo systemctl daemon-reload
```

Started the service:

```bash
sudo systemctl start day3-demo
```

Verified:

```bash
systemctl status day3-demo --no-pager
```

Result:

```text
Active: active (running)
```

The service was successfully restored.

### Troubleshooting Workflow

The production-style troubleshooting sequence used in this lab was:

```text
Service failure
      |
      v
systemctl status
      |
      v
Identify 203/EXEC
      |
      v
journalctl -u service
      |
      v
Inspect ExecStart
      |
      v
Verify executable/path
      |
      v
Identify root cause
      |
      v
Correct configuration
      |
      v
systemctl daemon-reload
      |
      v
Start service
      |
      v
Verify service status
```

## Lab Cleanup

Because `day3-demo.service` was created only for training, it was stopped and disabled after completing the lab:

```bash
sudo systemctl disable --now day3-demo
```

Verified:

```bash
systemctl is-active day3-demo
systemctl is-enabled day3-demo
```

Final state:

```text
inactive
disabled
```

## Key Takeaways

- PID and PPID help identify process relationships.
- Process states help identify running, sleeping, stopped, blocked, and zombie processes.
- `SIGTERM` should normally be attempted before `SIGKILL`.
- `top`, `free`, `vmstat`, and `iostat` help distinguish CPU, memory, and I/O problems.
- `nice` and `renice` can adjust CPU scheduling priority without terminating a process.
- `systemctl` manages systemd units and services.
- A service being `active` and `enabled` are two different concepts.
- systemd socket activation can start services on demand.
- `journalctl` is an important tool for investigating systemd service failures.
- Changes to systemd unit files require `systemctl daemon-reload`.
- `203/EXEC` can indicate that systemd could not execute the configured `ExecStart` command.
- Production troubleshooting should focus on identifying and confirming the root cause before making changes.