# Helpful Tools on Linux Systems

## General

### Stay in Sudo

```
sudo sudo su
```

### Delete All Files Except One

```
find . -type f ! -name "file_to_keep" -delete
```

### Start the SSH Agent (if not already running)

```
eval "$(ssh-agent -s)"
```

## Networking

### DNS Lookup

```
nslookup fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal
nslookup -query=AAAA fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal
```

Or using `dig`:
```
dig +short fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal A
dig +short fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal AAAA
```

### Checking Port Connectivity via Netcat

IPv4
```
nc -4zv fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal 2049
```

IPv6
```
nc -6zv fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal 2049
```

Or using `telnet` as a fallback:
```
telnet fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal 2049
```

### Show Current Network Connections

```
ss -tulnp
```

Or for specific port:
```
ss -ltnp 'sport = :2049'
```

### Checking IP Packet Rules

```
ip6tables -S
iptables -S
```

### Checking Connectivity via Ping

```
ping6 -c 1 fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal
```

### Report RPC Information about a Host

```
rpcinfo -p fs-09b63558bcee5dd19.fsx.us-east-1.aws.internal
```

### Generate a TCP Dump

Consider using WireShark to investigate the `.pcap` files

```
tcpdump -i any -nn -w single_nfs.pcap \
  '(tcp and src host 10.0.30.65 and src port 761 and dst host 10.0.22.54 and dst port 2049) \
   or (tcp and src host 10.0.22.54 and src port 2049 and dst host 10.0.30.65 and dst port 761)'
```

Or a more general filter:
```
tcpdump -i any -nn 'tcp and ((src port 761 and dst port 2049) or (src port 2049 and dst port 761))'
```

## Data

Use dd to write 10GB of output file (small block vs large block)

```
dd if=/dev/zero of=output.file bs=1G count=10
dd if=/dev/zero of=output.file bs=1M count=10240
```

Checking disk usage

```
df -h  # disk usage
df -i  # inode usage
```

Checking disk space used

```
du -h
```

The `df` command provides an estimate for how much space is being utilized on your filesystem. The `du` command is a much more accurate snapshot of a given directory or subdirectory. 

A live example is if you get an error trying to install somthing in `/var` that says the directory is full, you can run the `df` command to confirm that the directory is full. After validating this, you can run `du` through subdirectories to find the culprit.

## NFS

### Mount over NFSv4 (IPv4)

```
mount -t nfs -o proto=tcp,nfsvers=4.1,rsize=1048576,wsize=1048576,timeo=600 fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal:/fsx/ /tmp/fsx
```

### Mount over NFSv4 (IPv6)

```
mount -t nfs -o proto=tcp6,nfsvers=4.1,rsize=1048576,wsize=1048576,timeo=600 fs-0c5b9f6d0190494dd.fsx.us-east-1.aws.internal:/fsx/ /tmp/fsx
```

### Show NFS Statistics

Show only the client:
```
nfsstat -c
```

Show only the server:
```
nfsstat -s
```

## Debugging Processes

### Find Stack Traces of Threads

```
PID=2038
for tid in $(ls /proc/$PID/task); do
  echo "=== $tid ==="
  cat /proc/$PID/task/$tid/stack
done
```

### Show Open Files for a Process

```
lsof -p <pid>
```

### Attach strace to a Running Process

```
strace -p <pid> -s 100 -f
```

### Show Which Process Is Using a Port

```
lsof -i :2049
```

### System Resource Monitoring

```
top
```

## Performance

Consider you see that processes are being OOM-killed. You can check the system logs via:

```
dmesg | grep -i "out of memory"
journalctl -k | grep -i "oom"
```

View free RAM:

```
free -m  # shows memory in MB
top      # press M to sort by memory usage
```

Add temporary swap (it's best to use a fast, local SSD or low-latency cloud SSD volumes):

```
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

Creating swap may hide underlying memory leaks. If you still see the memory growing, remove the swap space via:

```
sudo swapoff /swapfile && sudo rm /swapfile
```

The following sections were taken from __Systems Performance: Enterprise and the Cloud__ by Brendan Gregg

When debugging issues constraining CPUs, the consider checking the following (Linux):

* `uptime`/`top`: Check the load averages to see if load is increasing or decreasing over time. Bear
this in mind when using the following tools, as load may be changing during your analysis.
* `vmstat`: Run vmstat(1) with a one-second interval and check the system-wide CPU utili-
zation (“us” + “sy”). Utilization approaching 100% increases the likelihood of scheduler
latency.
* `mpstat`: Examine statistics per-CPU and check for individual hot (busy) CPUs, identifying
a possible thread scalability problem.
* `top`: See which processes and users are the top CPU consumers.
* `pidstat`: Break down the top CPU consumers into user- and system-time.
* `perf`/`profile`: Profile CPU usage stack traces for both user- or kernel-time, to identify why
the CPUs are in use.
* `perf`: Measure IPC as an indicator of cycle-based inefficiencies.
* `showboost`/`turboboost`: Check the current CPU clock rates, in case they are unusually low.
* `dmesg`: Check for CPU temperature stall messages (“cpu clock throttled”).

uptime(1) is one of several commands that print the system load averages:

```
$ uptime
9:04pm up 268 day(s), 10:16, 2 users, load average: 7.76, 8.32, 8.60
```

The last three numbers are the 1-, 5-, and 15-minute load averages. By comparing the three numbers, you can determine if the load is increasing, decreasing, or steady during the last 15 minutes
(or so). This can be useful to know: if you are responding to a production performance issue and find that the load is decreasing, you may have missed the issue; if the load is increasing, the issue may be getting worse!

The virtual memory statistics command, vmstat(8), prints system-wide CPU averages in the last few columns, and a count of runnable threads in the first column. Here is example output from the Linux version:

Sample sizes every 1 second
```
[root /]# vmstat 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 4348244  56692 2060040    0    0   139    19   87  129  9  6 86  0  0
 0  0      0 4348216  56692 2060064    0    0     0     4 2556 4754  4  4 92  0  0
 1  0      0 4347588  56692 2060056    0    0     0     4 3039 5254  5  4 91  0  0
 0  0      0 4347952  56692 2060064    0    0     0     0 3569 5798  6  6 87  0  0
[...]
```

The first line of output is supposed to be the summary-since-boot. However, on Linux the procs and memory columns begin by showing the current state. CPU-related columns are:
- `r`: Run-queue length—the total number of runnable threads
- `us`: User-time percent
- `sy`: System-time (kernel) percent
- `id`: Idle percent
- `wa`: Wait I/O percent, which measures CPU idle when threads are blocked on disk I/O
- `st`: Stolen percent, which for virtualized environments shows CPU time spent servicing other tenants

All of these values are system-wide averages across all CPUs, with the exception of r, which is the total. On Linux, the r column is the total number of tasks waiting plus those running.

# Debugging Scenarios

You're dropped onto a box and you want to discover what the critical applications running are:

#### Look at Running Processes

Try one or both of the following:

```
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -20
```

```
top
```

The “core app” will usually be the long-running process (or set of processes) consuming the most CPU/memory or owned by a special service account.

#### Check Which Processes are Supervising oOthers:

```
pstree -ap
```

Example output from me running `pstree` from a shell:

```
systemd,1 --switched-root --system --deserialize 21
...
  ├─sshd,5768 -D
  │   └─sshd,3850
  │       └─sshd,3898
  │           └─bash,3899
  │               └─sudo,4346 sudo su
  │                   └─sudo,4347 su
  │                       └─su,4351
  │                           └─bash,4357
  │                               └─pstree,19211 -ap
  ```

#### Check `systemd` Services:

```
systemctl list-units --type=service --state=running
```

Look for non-standard services (not `sshd`, `cron`, etc.), e.g. `myapp.service`, `nginx.service`, `postgresql.service`.

#### See what’s Binding to the Network:

```
ss -lntp
```

Example output showing some process and its `pid`:

```
State     Recv-Q    Send-Q            Local Address:Port        Peer Address:Port   Process
LISTEN    0         128                     0.0.0.0:22               0.0.0.0:*       users:(("sshd",pid=4321,fd=2))
LISTEN    0         100                   127.0.0.0:25               0.0.0.0:*       users:(("java",pid=1234,fd=12))
...
```


