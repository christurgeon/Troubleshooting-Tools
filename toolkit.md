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
