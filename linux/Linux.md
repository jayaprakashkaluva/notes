# Linux for Software Engineers

## Must-Know Guide for Building and Running Production Applications

> **Goal:** Develop the Linux mental models needed to design, deploy, troubleshoot, secure, and operate production applications. The objective is not to become a Linux administrator; it is to understand what the OS is doing underneath your application well enough to make strong architecture and production decisions.

---

## 1. What a Software Engineer Should Be Able to Do

A  Software Engineer running applications on Linux should be able to:

- Explain what happens from starting a process until it executes application code.
- Diagnose whether a production problem is caused by the application, CPU, memory, disk, filesystem, network, DNS, TLS, kernel, container limits, or an external dependency.
- Understand how processes, threads, sockets, files, and containers map to Linux kernel resources.
- Reason about application resource limits and isolation.
- Understand what Docker/Podman and Kubernetes ultimately ask the Linux kernel to do.
- Diagnose high CPU, memory pressure, OOM kills, disk latency, connection exhaustion, file-descriptor exhaustion, DNS failures, and hung processes.
- Design applications that start, stop, log, expose health signals, and handle failures correctly on Linux.
- Communicate effectively with SRE, platform, networking, security, and infrastructure engineers.

---

# Part I — Linux Mental Model

## 2. Kernel Space vs User Space

### Why it matters

Your Java, Go, Python, Node.js, database, proxy, and container runtime normally execute in **user space**. Hardware and privileged resources are controlled by the **Linux kernel**.

### Mental model

```text
Application
    |
Libraries / Runtime (JVM, glibc, etc.)
    |
System Calls
    |
Linux Kernel
    |
CPU | Memory | Disk | Network | Devices
```

Applications cannot directly perform operations such as reading a disk block or sending a TCP packet. They request kernel services through **system calls**.

Know these terms:

- kernel space
- user space
- system calls
- interrupts
- context switches
- kernel modules
- device drivers

Useful commands:

```bash
uname -a
cat /etc/os-release
cat /proc/version
lsmod
dmesg
```

### engineer-level connection

When an API is slow, distinguish application execution time from time spent waiting inside kernel operations such as network I/O, disk I/O, locks, page faults, or scheduling.

---

## 3. System Calls

Important system calls to recognize:

```text
open / openat
read
write
close
socket
connect
accept
send / recv
epoll_wait
mmap
fork / clone
execve
futex
fsync
```

### Essential tool: `strace`

```bash
strace ./myapp
strace -p <PID>
strace -f -p <PID>
strace -c -p <PID>
```

Typical uses:

- Why is a process hanging?
- Which file is it trying to open?
- Is it repeatedly retrying a network connection?
- Is it blocked in `futex`, `poll`, `epoll_wait`, `read`, or `connect`?

Example:

```text
connect(...) = -1 ETIMEDOUT
```

This immediately changes the investigation from “the application is slow” to “a network connection attempt is timing out.”

---

# Part II — Processes, Threads, and Scheduling

## 4. Processes

Know:

- PID
- PPID
- process tree
- process states
- zombie processes
- orphan processes
- foreground/background processes
- process exit codes

Commands:

```bash
ps aux
ps -ef
pstree
top
pgrep java
cat /proc/<PID>/status
```

Important process states include:

```text
R  running/runnable
S  interruptible sleep
D  uninterruptible sleep, commonly waiting on I/O
T  stopped
Z  zombie
```

A large number of processes in `D` state can indicate storage or other kernel I/O problems.

---

## 5. Threads

Linux schedules **tasks**; application threads are visible to the kernel and scheduler.

Commands:

```bash
ps -eLf
ps -T -p <PID>
top -H -p <PID>
```

For JVM applications, this becomes extremely useful when correlating a hot Linux thread with a Java thread dump.

### Production example

```bash
top -H -p <java-pid>
```

Find the thread consuming CPU, convert its thread ID to hexadecimal if needed, and correlate it with `jstack` output.

---

## 6. Signals and Graceful Shutdown

Must know:

```text
SIGTERM
SIGKILL
SIGINT
SIGHUP
SIGQUIT
SIGCHLD
```

Commands:

```bash
kill <PID>
kill -TERM <PID>
kill -KILL <PID>
```

### Critical distinction

`SIGTERM` can be handled by the application and should normally initiate graceful shutdown.

`SIGKILL` cannot be caught or handled.

### Kubernetes connection

Pod termination roughly involves:

```text
Pod termination requested
        ↓
SIGTERM sent to container process
        ↓
termination grace period
        ↓
SIGKILL if process is still running
```

Applications should therefore correctly handle SIGTERM, stop accepting new work, finish or safely abandon in-flight work, release resources, and exit within the configured grace period.

---

## 7. CPU Scheduling

Understand:

- runnable tasks
- run queue
- scheduler
- CPU cores
- context switches
- CPU affinity
- nice values
- CPU throttling

Commands:

```bash
top
htop
mpstat -P ALL 1
pidstat 1
vmstat 1
uptime
```

### Load average

```bash
uptime
```

Example:

```text
load average: 8.2, 7.9, 6.5
```

Do **not** interpret load average as CPU percentage. Linux load includes runnable tasks and tasks in certain uninterruptible waits.

### engineer-level questions

When CPU is high, ask:

1. Is CPU actually saturated?
2. User CPU or system CPU?
3. One core or all cores?
4. Is the process being throttled by a cgroup?
5. Are context switches excessive?
6. Is the application busy doing useful work, GC, serialization, encryption, retries, or spinning?

---

# Part III — Memory

## 8. Virtual Memory

Every process operates in a **virtual address space**.

Conceptual model:

```text
Process virtual address
        ↓
Page tables
        ↓
Physical memory pages
```

Know:

- virtual memory
- physical memory
- pages
- page tables
- minor/major page faults
- resident set size (RSS)
- shared memory
- memory mapping (`mmap`)

Commands:

```bash
free -h
cat /proc/meminfo
cat /proc/<PID>/status
pmap <PID>
vmstat 1
```

---

## 9. Page Cache

Linux intentionally uses otherwise-free RAM to cache filesystem data.

Therefore:

```text
low "free" memory != necessarily a memory problem
```

Use `available` memory and overall pressure rather than looking only at `free`.

This is important when diagnosing databases, JVM services, and file-heavy workloads.

---

## 10. Swap

Understand what swap is and why active swapping can severely hurt latency-sensitive services.

Commands:

```bash
swapon --show
free -h
vmstat 1
```

In `vmstat`, pay attention to:

```text
si  swap in
so  swap out
```

---

## 11. OOM Killer

When memory cannot be satisfied, Linux may kill processes.

Investigate with:

```bash
dmesg | grep -i oom
journalctl -k | grep -i oom
cat /proc/<PID>/oom_score
```

### Container connection

A process can be OOM-killed because its **container/cgroup memory limit** is exhausted even when the host still has memory available.

This distinction is essential in Kubernetes.

---

## 12. Memory Pressure Questions

When a service has memory problems, investigate:

```text
Application heap
Native/off-heap memory
Thread stacks
Memory-mapped files
Page cache
Container memory limit
Kernel memory
Swap activity
Memory leaks
GC behavior
```

For a JVM, do not assume:

```text
-Xmx == total process memory
```

The JVM also consumes metaspace, code cache, thread stacks, direct buffers, native libraries, GC structures, and other native memory.

---

# Part IV — Files, File Descriptors, and Filesystems

## 13. File Descriptors

Linux represents many resources through file descriptors:

```text
regular files
TCP sockets
UDP sockets
pipes
terminals
event descriptors
```

Typical descriptors:

```text
0 stdin
1 stdout
2 stderr
```

Commands:

```bash
lsof -p <PID>
ls -l /proc/<PID>/fd
ulimit -n
cat /proc/sys/fs/file-max
```

### Production failure

```text
Too many open files
```

Possible causes:

- socket leak
- file leak
- excessively large connection pool
- insufficient process limit

Never fix this blindly by only increasing the limit; determine why descriptors are being consumed.

---

## 14. VFS, Inodes, and Dentries

Mental model:

```text
Application
   ↓
VFS (Virtual File System)
   ↓
ext4 / XFS / NFS / overlayfs / ...
   ↓
Block layer / remote filesystem
```

Understand:

- inode = filesystem metadata object
- directory entries map names to inodes
- filename is not the file's inode
- hard links reference the same inode
- symbolic links reference another pathname

Commands:

```bash
ls -li
stat file
findmnt
df -h
df -i
du -sh *
```

### Important production issue

A filesystem can have free bytes but run out of **inodes**:

```bash
df -i
```

Applications then fail to create files despite apparent disk capacity.

---

## 15. Deleted Files Still Consuming Disk

A classic production issue:

```bash
rm huge.log
```

Disk space may not return if a process still holds the file descriptor open.

Find it with:

```bash
lsof +L1
```

This is a must-know operational pattern.

---

## 16. Mounts and Filesystems

Commands:

```bash
mount
findmnt
lsblk
blkid
df -h
```

Understand:

- mount points
- local vs network filesystems
- read-only mounts
- bind mounts
- container volume mounts
- persistent volumes

---

# Part V — Storage and I/O

## 17. Linux Storage Path

Conceptual path:

```text
Application
    ↓
Filesystem
    ↓
Page Cache
    ↓
Block I/O layer
    ↓
Device driver
    ↓
SSD / HDD / virtual/cloud block device
```

Understand:

- buffered I/O
- direct I/O
- sequential vs random I/O
- IOPS
- throughput
- latency
- queue depth
- `fsync`

---

## 18. `fsync` and Durability

A successful application `write()` does not always mean the bytes are durable on physical storage.

Durability-sensitive systems—databases, logs, queues—often rely on `fsync` or equivalent mechanisms.

engineer-level connection:

```text
performance ↔ durability trade-off
```

This matters when reasoning about database WALs, Kafka logs, transactional storage, and crash consistency.

---

## 19. Disk Troubleshooting

Commands:

```bash
iostat -xz 1
pidstat -d 1
vmstat 1
df -h
du -sh
```

Questions:

- Is the disk full?
- Are inodes exhausted?
- Is latency high?
- Which process generates the I/O?
- Is the process blocked in `D` state?
- Is a cloud disk hitting IOPS/throughput limits?
- Is logging unexpectedly generating huge write volume?

---

# Part VI — Networking

## 20. Linux Networking Mental Model

```text
Application
    ↓
Socket
    ↓
TCP / UDP
    ↓
IP
    ↓
Routing
    ↓
Network interface
    ↓
Network
```

Staff engineers should be comfortable debugging every stage.

---

## 21. Sockets

A server roughly does:

```text
socket()
bind()
listen()
accept()
read()/write()
```

A client roughly does:

```text
socket()
connect()
read()/write()
```

Commands:

```bash
ss -lntp
ss -tan
ss -s
lsof -i
```

Know TCP states such as:

```text
LISTEN
SYN-SENT
SYN-RECEIVED
ESTABLISHED
FIN-WAIT
CLOSE-WAIT
TIME-WAIT
```

A large number of `CLOSE-WAIT` connections often indicates the application is not closing sockets after the remote peer closes them.

---

## 22. IP Addresses, Interfaces, and Routes

Commands:

```bash
ip addr
ip link
ip route
ip neigh
```

Understand:

- loopback
- interface addresses
- subnet
- default gateway
- route selection
- ARP/neighbor discovery at a practical level

---

## 23. DNS

Commands:

```bash
dig example.com
nslookup example.com
getent hosts example.com
cat /etc/resolv.conf
```

Application symptoms of DNS problems can look like generic network latency or connection failures.

Know that application name resolution may involve libc/runtime behavior, resolver configuration, search domains, caches, Kubernetes DNS, and upstream DNS servers.

---

## 24. HTTP and TLS Debugging

Commands:

```bash
curl -v https://example.com
curl -I https://example.com
openssl s_client -connect example.com:443 -servername example.com
```

Use these to distinguish:

```text
DNS failure
TCP connection failure
TLS handshake failure
HTTP/application failure
```

---

## 25. Packet Capture

`tcpdump` is one of the highest-value Linux troubleshooting tools.

Examples:

```bash
sudo tcpdump -i any port 443
sudo tcpdump -i any host 10.0.0.5
sudo tcpdump -nn -i any tcp
```

Use it to answer:

- Did the request leave the host?
- Did SYN receive SYN-ACK?
- Are retransmissions occurring?
- Is the remote side closing the connection?
- Is DNS traffic leaving the machine?

---

## 26. NAT, Firewall, and Connection Tracking

Know conceptually:

- NAT
- SNAT
- DNAT
- firewall rules
- connection tracking (`conntrack`)
- ephemeral ports

Tools may include:

```bash
nft
iptables
sysctl
```

This knowledge is particularly useful for Kubernetes networking, egress, load balancers, service meshes, and high-connection-count applications.

---

## 27. Ephemeral Port Exhaustion

A high-volume client can run out of local ephemeral ports.

Inspect:

```bash
sysctl net.ipv4.ip_local_port_range
ss -s
```

engineer-level lesson: a “network failure” can actually be local resource exhaustion caused by connection behavior.

---

# Part VII — Containers Are Linux

## 28. Namespaces

Linux namespaces provide isolation of resources such as:

```text
PID
network
mount
UTS/hostname
IPC
user
cgroup
```

Conceptually:

```text
Container = isolated process view + constrained resources + filesystem + runtime configuration
```

Containers are **not lightweight virtual machines**. They are processes using Linux isolation primitives.

Useful command:

```bash
lsns
```

---

## 29. cgroups

Control groups manage/account for resources such as:

```text
CPU
memory
I/O
process counts
```

This is the foundation behind Kubernetes resource limits.

Example:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

The kernel ultimately enforces these constraints using Linux mechanisms including cgroups.

Understand:

- CPU quota/throttling
- memory limits
- OOM behavior
- cgroup v2

Commands worth exploring:

```bash
mount | grep cgroup
cat /proc/<PID>/cgroup
systemd-cgls
systemd-cgtop
```

---

## 30. Container Filesystems

Understand conceptually:

- image layers
- copy-on-write
- overlayfs
- writable container layer
- bind mounts
- volumes

A typical conceptual stack:

```text
Writable container layer
Image layer N
Image layer N-1
...
Base image
```

Do not treat the container writable layer as durable application storage.

---

## 31. Kubernetes Linux Foundations

Map Kubernetes concepts back to Linux:

| Kubernetes concept | Linux foundation |
|---|---|
| Container | process + namespaces + cgroups + filesystem |
| CPU limit | cgroup CPU control |
| Memory limit | cgroup memory control |
| Pod IP | Linux networking / network namespace / CNI |
| Volume | mounts/filesystems |
| Secret/config file | filesystem/mount mechanisms |
| Pod termination | Unix signals + process lifecycle |
| OOMKilled | memory/cgroup/kernel behavior |
| Node pressure | Linux resource pressure |

A software engineer should be able to debug below the Kubernetes abstraction when necessary.

---

# Part VIII — Service Management and Boot

## 32. systemd

On many Linux distributions, systemd manages services.

Commands:

```bash
systemctl status myservice
systemctl start myservice
systemctl stop myservice
systemctl restart myservice
systemctl enable myservice
```

Example unit concepts:

```ini
[Service]
ExecStart=/opt/app/myapp
Restart=on-failure
```

Understand:

- service dependencies
- restart policies
- environment configuration
- service users
- resource controls

---

## 33. journald

Commands:

```bash
journalctl
journalctl -u myservice
journalctl -f
journalctl --since "10 minutes ago"
journalctl -k
```

Know where your application logs go and how log rotation/retention is managed.

---

# Part IX — Users, Permissions, and Security

## 34. Users and Groups

Commands:

```bash
id
whoami
groups
```

Understand UID and GID. Containers also ultimately execute processes under user identities recognized by the kernel.

---

## 35. File Permissions

Understand:

```text
r = read
w = write
x = execute
```

and:

```text
owner / group / others
```

Commands:

```bash
ls -l
chmod
chown
chgrp
```

Know numeric modes such as:

```text
755
644
600
```

Also understand directory permissions, which differ subtly from file permissions.

---

## 36. Linux Capabilities

Traditional root privileges can be split into capabilities.

Examples include:

```text
CAP_NET_BIND_SERVICE
CAP_NET_ADMIN
CAP_SYS_ADMIN
```

Commands:

```bash
getcap
setcap
capsh --print
```

engineer-level principle: prefer the minimum privilege required rather than running application containers as root.

---

## 37. SELinux, AppArmor, and seccomp

You do not need to be a security specialist, but you should understand their roles:

- **SELinux/AppArmor:** mandatory access control.
- **seccomp:** restricts which system calls a process may use.
- **Linux capabilities:** split privileged operations into smaller units.

These controls can explain why something works locally but fails in a hardened production/container environment.

---

# Part X — Resource Limits

## 38. `ulimit` and Process Limits

Commands:

```bash
ulimit -a
ulimit -n
ulimit -u
```

Relevant limits include:

- open files
- process/thread counts
- stack size
- locked memory

Also inspect:

```bash
cat /proc/<PID>/limits
```

Production lesson: resource limits should match expected concurrency and failure behavior, but increasing them is not a substitute for fixing leaks or unbounded resource use.

---

# Part XI — Observability and Troubleshooting Tools

## 39. `/proc`

`/proc` exposes kernel and process information.

Important examples:

```bash
/proc/cpuinfo
/proc/meminfo
/proc/loadavg
/proc/net/
/proc/<PID>/status
/proc/<PID>/fd/
/proc/<PID>/limits
/proc/<PID>/maps
/proc/<PID>/cgroup
```

Think of `/proc` as one of your windows into the kernel's current view of the system.

---

## 40. `/sys`

`/sys` exposes kernel objects, devices, cgroups, and tunable information through sysfs.

You do not need to memorize its entire structure. Know what it represents and how platform/SRE tooling uses it.

---

## 41. Core Command Toolkit

### Process

```bash
ps
pstree
top
pidstat
pgrep
kill
```

### Memory

```bash
free
vmstat
pmap
```

### Files

```bash
ls
stat
find
lsof
df
du
findmnt
```

### Disk

```bash
lsblk
iostat
pidstat -d
```

### Network

```bash
ip
ss
dig
getent
curl
openssl
tcpdump
```

### Kernel/system

```bash
uname
dmesg
journalctl
sysctl
```

### Process internals

```bash
strace
perf
```

---

# Part XII — Production Debugging Playbooks

## 42. High CPU

Suggested investigation:

```text
1. Confirm host/container CPU saturation.
2. Identify process.
3. Identify hot threads.
4. Determine user vs system CPU.
5. Check cgroup throttling.
6. Correlate with application profiler/thread dump.
7. Look for loops, GC, serialization, compression, crypto, retries, lock contention, or traffic changes.
```

Commands:

```bash
top
mpstat -P ALL 1
pidstat -u 1
top -H -p <PID>
perf top
```

---

## 43. Memory / OOM

```text
1. Check host memory.
2. Check container/cgroup memory limit.
3. Check process RSS.
4. Check swap activity.
5. Check kernel/OOM logs.
6. Separate heap from native/off-heap memory.
7. Check workload/concurrency changes.
```

Commands:

```bash
free -h
vmstat 1
cat /proc/<PID>/status
dmesg | grep -i oom
```

For JVMs, combine Linux evidence with heap/native-memory/GC diagnostics.

---

## 44. Disk Full

```text
1. df -h
2. df -i
3. du to identify large directories
4. lsof +L1 for deleted-open files
5. inspect logs/temp files/container storage
```

Commands:

```bash
df -h
df -i
du -xhd1 /
lsof +L1
```

---

## 45. High Disk Latency

```text
1. Confirm I/O latency/utilization.
2. Identify process producing I/O.
3. Check D-state tasks.
4. Check device/cloud volume limits.
5. Determine read vs write pressure.
6. Inspect application fsync/logging/database behavior.
```

Commands:

```bash
iostat -xz 1
pidstat -d 1
vmstat 1
ps -eo state,pid,comm | grep '^D'
```

---

## 46. Network Latency / Connection Failure

Use a layered investigation:

```text
DNS
 ↓
Routing
 ↓
TCP connection
 ↓
TLS
 ↓
HTTP/application protocol
 ↓
Remote application
```

Commands:

```bash
dig host
ip route
ss -tan
curl -v https://host
openssl s_client -connect host:443 -servername host
tcpdump
```

Avoid immediately blaming “the network.” Identify the layer that fails.

---

## 47. Connection Exhaustion

Check:

```bash
ss -s
ss -tan
lsof -p <PID> | wc -l
ulimit -n
cat /proc/<PID>/limits
```

Look for:

- too many established connections
- `CLOSE-WAIT`
- excessive `TIME-WAIT`
- file descriptor exhaustion
- connection pool misconfiguration
- ephemeral port pressure
- downstream slowness causing connections to accumulate

---

## 48. Process Hangs

Investigate:

```bash
ps -o pid,state,wchan,cmd -p <PID>
strace -f -p <PID>
top -H -p <PID>
```

Then correlate with application-specific diagnostics such as JVM thread dumps.

Potential causes:

```text
lock contention
deadlock
blocked disk I/O
network read/connect
futex wait
thread-pool starvation
downstream dependency stall
```

---

## 49. DNS Failure

```bash
cat /etc/resolv.conf
getent hosts service.example.com
dig service.example.com
```

For Kubernetes, additionally reason about pod DNS configuration, service names, search domains, CoreDNS, network policy, and upstream resolvers.

---

## 50. TLS Failure

```bash
curl -v https://service
openssl s_client -connect service:443 -servername service
```

Investigate:

- certificate expiration
- hostname/SAN mismatch
- trust chain
- TLS protocol/cipher mismatch
- SNI
- proxy/service-mesh interception
- clock problems

---

# Part XIII — JVM on Linux

## 51. Linux Knowledge Every Java Engineer Should Connect to the JVM

Map JVM concepts to OS resources:

```text
Java thread       → Linux schedulable thread/task
Socket            → Linux file descriptor
File              → Linux file descriptor
Heap pages         → process virtual/physical memory
DirectByteBuffer   → native/off-heap memory
GC CPU             → Linux CPU scheduling
-Xmx               → only part of total process memory
Container limit    → cgroup memory/CPU constraints
SIGTERM            → graceful application shutdown
```

Useful combined toolkit:

```bash
top -H -p <PID>
jstack <PID>
jcmd <PID> VM.native_memory summary
jcmd <PID> GC.heap_info
lsof -p <PID>
cat /proc/<PID>/status
cat /proc/<PID>/limits
```

The exact JVM commands available depend on JVM version/configuration.

---

# Part XIV — Executables, Libraries, and Shell Environment

## 52. Program Execution

Understand conceptually what occurs around:

```bash
./app
```

Linux loads an executable, maps required memory/libraries, establishes process state, and transfers execution to the program's entry path.

Useful tools:

```bash
file <binary>
ldd <binary>
readelf -h <binary>
which <command>
type <command>
```

Know the difference between:

- statically linked binary
- dynamically linked binary
- interpreter/script
- JVM bytecode executed by a runtime

---

## 53. Shared Libraries and Dynamic Linking

A production binary may fail because a required shared library is unavailable or incompatible.

Useful commands:

```bash
ldd ./binary
ldconfig -p
```

Understand at a high level:

```text
ELF
shared objects (.so)
dynamic linker
library search paths
ABI compatibility
```

---

## 54. Shell and Environment

Must know practical shell behavior:

```bash
export NAME=value
env
printenv
echo $PATH
command -v java
```

Understand:

- environment variables are inherited by child processes
- PATH determines executable lookup
- quoting matters
- exit status matters
- pipes connect process streams
- redirection controls stdin/stdout/stderr

Examples:

```bash
command1 | command2
app > app.log 2>&1
echo $?
```

For production applications, avoid putting secrets casually in shell history, command lines, or logs.

---

# Part XV — Time

## 55. Clocks, NTP, and Time Zones

Distributed systems depend on sane clocks.

Commands:

```bash
date
timedatectl
```

Understand:

- UTC vs local time
- timezone configuration
- clock synchronization/NTP
- wall clock vs monotonic clock

engineer-level connection:

Clock issues can affect:

```text
TLS certificates
token expiration
logs/event ordering
leases/timeouts
scheduled jobs
metrics
```

Use monotonic time sources for measuring durations where your language/runtime provides them.

---

# Part XVI — Kernel Configuration

## 56. `sysctl`

Inspect kernel runtime parameters:

```bash
sysctl -a
sysctl net.ipv4.ip_local_port_range
sysctl vm.swappiness
```

Areas relevant to application engineering can include:

```text
network queues
TCP behavior
ephemeral ports
file limits
virtual memory
```

Do not cargo-cult production `sysctl` settings. Understand the bottleneck and operating environment before changing kernel parameters.

---

# Part XVII — Advanced Topics Worth Knowing

## 57. NUMA

On large multi-socket systems, memory access cost can depend on which CPU/socket owns the memory.

Commands:

```bash
numactl --hardware
lscpu
```

Relevant for large JVMs, databases, caches, and high-performance workloads—not something every ordinary microservice requires tuning for.

---

## 58. Huge Pages

Linux normally manages memory in relatively small pages; huge pages can reduce page-table/TLB overhead for some large-memory workloads.

Know the concept and why databases/JVMs may discuss huge pages or transparent huge pages. Tune only with workload-specific evidence.

---

## 59. `perf`

For advanced CPU analysis:

```bash
perf top
perf stat <command>
perf record <command>
perf report
```

It can help identify where CPU cycles, cache misses, context switches, and other low-level costs occur.

---

## 60. eBPF

At Staff level, understand what eBPF enables even if you are not yet writing eBPF programs:

- kernel/application observability
- networking visibility
- tracing
- profiling
- security enforcement

Modern observability and Kubernetes networking tools increasingly build on eBPF.

---

# Part XVIII — SSH and Remote Operations

## 61. SSH Basics

Commands:

```bash
ssh user@host
scp file user@host:/path
ssh-keygen
```

Understand:

- public/private keys
- host keys
- known_hosts
- SSH agent
- port forwarding at a conceptual level

Production access should follow organizational security controls; avoid sharing private keys or disabling host verification casually.

---

# Part XIX — Reliability Principles

## 62. Build Linux-Friendly Applications

Production services should:

- handle SIGTERM gracefully
- have bounded thread/connection/work queues
- set sensible timeouts
- avoid file/socket leaks
- avoid uncontrolled retry storms
- expose health and observability signals
- write logs to the platform-supported logging path
- avoid depending on writable local container storage for durable state
- run with least privilege
- respect container CPU/memory constraints
- use UTC consistently for distributed data where appropriate
- make resource consumption measurable

---

# Part XX — Prioritized Learning Roadmap

## Level 1 — MUST KNOW

Master these first:

1. Kernel vs user space
2. Processes and threads
3. Signals and graceful shutdown
4. System calls and `strace`
5. CPU scheduling and load
6. Virtual memory, RSS, page cache, swap
7. OOM killer and container OOM
8. File descriptors and limits
9. Filesystems, inodes, mounts, permissions
10. Disk capacity vs inode exhaustion
11. TCP sockets and connection states
12. IP addresses and routing
13. DNS
14. HTTP/TLS debugging
15. `tcpdump`
16. namespaces
17. cgroups
18. container filesystem basics
19. systemd/journald basics
20. `/proc`
21. `top`, `ps`, `vmstat`, `iostat`, `pidstat`, `lsof`, `ss`, `ip`, `curl`, `dig`
22. JVM ↔ Linux resource mapping if you run Java

### Outcome

You should be able to diagnose most common application-level production incidents without treating Linux as a black box.

---

## Level 2 — SHOULD KNOW

1. VFS
2. storage I/O path
3. `fsync` and durability
4. page faults
5. dynamic linking
6. Linux capabilities
7. seccomp
8. SELinux/AppArmor concepts
9. NAT/conntrack
10. ephemeral ports
11. kernel parameters/sysctl
12. shell/process environment
13. time synchronization
14. overlayfs
15. cgroup v2 details
16. service startup/dependency behavior

### Outcome

You can reason across application, container, host, and infrastructure boundaries.

---

## Level 3 — ADVANCED / LEARN AS NEEDED

1. `perf`
2. eBPF/BCC/bpftrace concepts and tooling
3. NUMA
4. huge pages
5. advanced TCP tuning
6. scheduler internals
7. filesystem internals
8. block I/O scheduler internals
9. kernel tracing
10. network namespaces/veth/bridges in depth
11. container runtime internals

### Outcome

You can participate effectively in deep performance, platform, and kernel-adjacent investigations.

---

# Part XXI — A software engineer's Production Checklist

When a Linux-hosted application is unhealthy, walk through the system rather than guessing.

```text
APPLICATION
│
├── Is the process alive?
├── Is it making progress?
├── Are threads blocked?
├── Are queues/pools exhausted?
│
CPU
│
├── Saturated?
├── Hot threads?
├── System CPU?
├── cgroup throttling?
│
MEMORY
│
├── Host pressure?
├── Container limit?
├── RSS growing?
├── Swap?
├── OOM kill?
│
FILES / STORAGE
│
├── Disk full?
├── Inodes full?
├── Deleted-open files?
├── I/O latency?
├── D-state tasks?
│
NETWORK
│
├── DNS?
├── Route?
├── TCP connection?
├── TLS?
├── Connection/file descriptor exhaustion?
├── Packet loss/retransmission?
│
DEPENDENCIES
│
├── Database?
├── Kafka/queue?
├── Cache?
├── External API?
│
PLATFORM
│
├── cgroups/resources?
├── Container filesystem/mount?
├── Kubernetes events/probes?
├── Security policy?
└── Node/kernel problem?
```

---

# Part XXII — System Design and Interview Connections

Linux knowledge strengthens system-design reasoning because many “distributed system” limits eventually become OS limits.

| Design topic | Linux knowledge that matters |
|---|---|
| High-concurrency API | sockets, FDs, epoll, threads, CPU |
| Kafka/streaming | page cache, disk I/O, fsync, networking |
| Database | memory, page cache, I/O, fsync, filesystems |
| Kubernetes | namespaces, cgroups, signals, networking |
| WebSockets | sockets, FDs, TCP states, memory |
| Reverse proxy | sockets, ports, queues, TLS |
| JVM service | threads, virtual memory, cgroups, signals |
| Large cache | memory, NUMA, page behavior |
| Low-latency system | scheduling, page faults, I/O, networking |
| Durable queue | filesystem, write buffering, fsync |
| Service mesh | sockets, routing, proxies, TLS, namespaces |

### Questions you should be able to reason through

- Why can a server with plenty of RAM still OOM-kill a container?
- Why can deleting a log file fail to free disk space?
- Why can a process fail with “too many open files”?
- Why can load average be high while CPU utilization is not 100%?
- Why can `write()` succeed before data is durable?
- Why can a JVM use much more memory than `-Xmx`?
- Why can a service have thousands of `CLOSE-WAIT` sockets?
- Why can a client fail to open new connections even though the server is healthy?
- What actually happens when Kubernetes terminates a pod?
- What Linux mechanisms make a container a container?
- How would you prove whether latency is in DNS, TCP, TLS, the application, or storage?

---

# Part XXIII — Suggested Hands-On Lab

Reading Linux concepts is not enough. Build a small lab and deliberately create failures.

## Exercise 1 — Processes

Start an application and inspect:

```bash
ps
pstree
/proc/<PID>
top -H
```

Send SIGTERM and SIGKILL and observe the difference.

## Exercise 2 — File descriptors

Open many sockets/files and inspect:

```bash
lsof -p <PID>
/proc/<PID>/fd
```

## Exercise 3 — Memory

Run a memory-consuming process and observe:

```bash
free -h
vmstat 1
/proc/<PID>/status
```

Then repeat inside a memory-limited container.

## Exercise 4 — Network

Run a local HTTP server and inspect:

```bash
ss -lntp
curl -v
tcpdump
```

## Exercise 5 — Container internals

Run a container and compare host/container views of:

```text
process IDs
network interfaces
mounts
cgroups
hostname
```

## Exercise 6 — Disk

Generate I/O and inspect:

```bash
iostat -xz 1
pidstat -d 1
```

Create many tiny files and observe inode consumption.

## Exercise 7 — JVM

Run a Spring Boot application and correlate:

```text
Linux PID
Linux threads
Java threads
file descriptors
heap
native memory
cgroup limits
```

This is one of the most valuable exercises for a Java-focused software engineer.

---

# Final Mental Model

The most important model to retain is:

```text
Your application
      │
      ├── processes / threads
      ├── virtual memory
      ├── file descriptors
      ├── files
      └── sockets
             │
             ▼
        System calls
             │
             ▼
        Linux Kernel
      ┌──────┼────────┐
      ▼      ▼        ▼
     CPU   Memory   Network
              │
              ▼
            Storage

Containers do not replace this model.
Kubernetes does not replace this model.
They add abstractions and orchestration around it.
```

A strong software engineer should be able to move **down the abstraction stack** when production behavior cannot be explained at the application or Kubernetes layer.

---

## Compact Command Cheat Sheet

```bash
# OS
uname -a
cat /etc/os-release

# Processes
ps -ef
pstree
top
top -H -p <PID>
pidstat 1

# Process internals
strace -f -p <PID>
cat /proc/<PID>/status
cat /proc/<PID>/limits
ls -l /proc/<PID>/fd

# CPU
mpstat -P ALL 1
vmstat 1

# Memory
free -h
cat /proc/meminfo
pmap <PID>

# Disk/filesystem
df -h
df -i
du -xhd1 <path>
lsblk
findmnt
iostat -xz 1
lsof +L1

# Network
ip addr
ip route
ss -lntp
ss -tan
ss -s
dig <host>
getent hosts <host>
curl -v <url>
openssl s_client -connect host:443 -servername host
sudo tcpdump -nn -i any

# Logs/kernel
journalctl
journalctl -u <service>
journalctl -k
dmesg

# Limits/cgroups
ulimit -a
cat /proc/<PID>/cgroup
systemd-cgls

# JVM examples
jstack <PID>
jcmd <PID> GC.heap_info
jcmd <PID> VM.native_memory summary
```

---

**Recommended learning order:** Processes → system calls → CPU → memory → file descriptors/filesystems → networking → storage → namespaces/cgroups/containers → systemd/security → performance tooling. Learn each topic by deliberately creating and diagnosing a failure.
