# Lab 16 — Containerization

**Roadmap topics:** ulimits · cgroups · Container Runtime · Docker
**Time:** about 5 hours. Split it: sections 1–4 (what a container is made of), then 5 (Docker itself).
**Touches system state:** yes, alongside real workloads, so the lab is strict about names. Everything it creates is named `lab-…` and carries the label `lab=16`; every published port is bound to `127.0.0.1`; cleanup removes things **by label and by name**. Existing containers, images, volumes and the Docker daemon's configuration are only inspected.

## Three rules on a machine with real containers

1. **Never run a blanket prune here.** `docker system prune`, `docker volume prune` and `docker container prune` act on everything that is not currently in use, including a stopped database container and its data. This lab never needs them.
2. **Never restart the Docker daemon as an experiment.** Without `live-restore` configured, restarting `dockerd` stops every container.
3. **Check the name before `docker rm`, `docker stop` and `docker volume rm`.** In this lab every such command names something starting with `lab-`.

## What you will be able to do

- Explain what a container is in terms of kernel features, and prove each part on a running container.
- Read and set per-process resource limits, and fix "Too many open files".
- Read the cgroup tree, impose memory, CPU and process limits, and recognise an OOM kill.
- Describe the chain from `docker run` down to the kernel, and name the alternatives at each level.
- Build an image, run it properly, persist its data, network it, compose it, limit and harden it, and clean up after it.

## Setup

```bash
mkdir -p ~/labwork/16 && cd ~/labwork/16
docker version --format 'client {{.Client.Version}}  server {{.Server.Version}}'
docker compose version
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}'      # what is already running: leave these alone
docker ps -a --filter label=lab=16 -q | wc -l                        # expect 0

docker pull alpine:3
docker run -d --name lab-sleep --label lab=16 alpine:3 sleep 86400   # a specimen to examine
```

If `docker` needs `sudo`, you are not in the `docker` group. That is a reasonable security posture (Lab 08: the group is root-equivalent); prefix the commands with `sudo`.

---

## 1. What a container is

A container is **an ordinary Linux process** that the kernel has been told to treat specially in three ways:

| Ingredient | Kernel feature | Controls |
|---|---|---|
| Isolation | **namespaces** | what the process can **see**: its own process list, network, mounts, hostname |
| Limits | **cgroups** | how much it can **use**: memory, CPU, process count, I/O |
| Filesystem | an **image** unpacked and mounted as its root (overlayfs) | which files it has |
| Restriction | capabilities, seccomp, SELinux | what it is **allowed to do** |

There is no "container" object in the kernel. A container runtime is a program that sets up those features and then starts your process. There is no second kernel either: a container shares the host's kernel, which is why it starts in milliseconds, and why it is a weaker boundary than a virtual machine.

### Prove it

```bash
docker top lab-sleep                              # the container's processes, as the HOST sees them
PID=$(docker inspect -f '{{.State.Pid}}' lab-sleep); echo "host PID: $PID"
ps -o pid,ppid,user,cmd -p "$PID"                 # an ordinary process on the host
docker exec lab-sleep ps                          # inside, the same process is PID 1
pstree -sp "$PID"                                 # its ancestry: systemd → containerd-shim → sleep
```

The same process has two PIDs because it lives in its own **PID namespace**.

### Namespaces

| Namespace | Isolates | Effect in a container |
|---|---|---|
| `pid` | process IDs | its first process is PID 1; it sees only its own processes |
| `net` | interfaces, addresses, routes, firewall rules, ports | its own `eth0` and `lo`; port 80 inside does not clash with the host's |
| `mnt` | mount points | its own root filesystem |
| `uts` | hostname | its own hostname |
| `ipc` | shared memory, semaphores | no IPC with other containers |
| `user` | UID and GID mappings | root inside can map to an unprivileged user outside (rootless mode) |
| `cgroup` | view of the cgroup tree | sees its own cgroup as the root |
| `time` | clock offsets | rarely used |

```bash
sudo ls -l /proc/$PID/ns/                         # one symlink per namespace, each with an ID
sudo ls -l /proc/$$/ns/                           # your shell's: different IDs for most of them
sudo lsns -p "$PID"                               # the namespaces of that process
sudo lsns -t net                                  # every network namespace on the host, with the process holding it
```

### A container by hand

`unshare` starts a program in new namespaces. No Docker involved:

```bash
sudo unshare --pid --fork --mount-proc --uts --net bash
```

You are now in a shell with its own PID, UTS and network namespaces:

```bash
hostname lab-by-hand; hostname                    # changed here only
ps -ef                                            # two processes. Your bash is PID 1
ip -br link                                       # only a loopback interface, and it is down
exit
hostname                                          # the host's name is untouched
```

Add a root filesystem from an image, a cgroup, and a virtual network cable, and that is a container. Lab 15's `labns` was the network-namespace part of this.

The reverse tool is `nsenter`, which joins a running process's namespaces. It is how you debug a container that has no shell or tools inside:

```bash
sudo nsenter -t "$PID" -n ip -br addr             # the container's network, using the HOST's ip command
sudo nsenter -t "$PID" -u hostname                # its hostname
sudo nsenter -t "$PID" -m ls /                    # its root filesystem
```

### The filesystem

```bash
docker inspect -f '{{.GraphDriver.Name}}' lab-sleep                 # overlay2
docker inspect -f '{{json .GraphDriver.Data}}' lab-sleep | tr ',' '\n'
docker exec lab-sleep touch /written-in-container
docker diff lab-sleep                             # what this container changed relative to its image
```

An image is a stack of read-only **layers**. A container adds one thin writable layer on top (`UpperDir`). Many containers share the same image layers, which is why a hundred containers from one image cost little disk. The writable layer is deleted with the container, which is why data that matters goes in a **volume** (section 5).

### Restrictions

```bash
grep -E 'Cap(Prm|Eff|Bnd)' /proc/$PID/status      # capability bitmasks: a reduced set, even though it runs as root
grep Seccomp /proc/$PID/status                    # 2 = a syscall filter is active
ps -Z -p "$PID"                                   # SELinux: container_t, with a per-container category pair
docker exec lab-sleep id                          # uid=0 inside
```

Root inside a default container is real root with most of its powers removed: a limited capability set, a syscall filter, and on Rocky an SELinux type (`container_t`) that confines it to its own files. Options such as `--privileged`, `--cap-add`, `--pid=host`, `-v /:/host` and mounting `/var/run/docker.sock` each remove part of that protection.

### Exercises 1

1. For `lab-sleep`: give its host PID, its PID inside the container, its parent process on the host, and its network namespace ID.
2. Pick one of your real running containers. Without `docker exec`, show its IP address and its listening sockets from the host. (`nsenter -n` with `ip` and `ss`.)
3. Start `sudo unshare --uts --pid --fork --mount-proc sleep 300 &`. Find it with `lsns`. Which namespaces does it share with your shell, and which are its own?
4. Is a container's root user the same UID as the host's root? What does that imply for a file created inside a bind-mounted host directory?
5. In one sentence each: how does a container differ from a VM in what is isolated, in start-up cost, and in the strength of the boundary?

<details><summary>Solutions</summary>

```bash
# 1
PID=$(docker inspect -f '{{.State.Pid}}' lab-sleep); echo "$PID"; docker exec lab-sleep ps -o pid,comm | head -3
ps -o ppid= -p "$PID" | xargs ps -o pid,cmd -p; sudo readlink /proc/$PID/ns/net

# 2
P=$(docker inspect -f '{{.State.Pid}}' YOUR_CONTAINER)
sudo nsenter -t "$P" -n ip -br addr; sudo nsenter -t "$P" -n ss -tlnp

# 3
sudo lsns | grep 'sleep 300'       # its own: pid, uts, mnt. Shared with the host: net, ipc, user, cgroup
sudo pkill -f 'sleep 300'
```

4. Yes, by default (no user namespace): UID 0 inside is UID 0 outside. A file it creates in a bind mount is owned by root on the host. This is the main argument for running container processes as a non-root user, and for rootless runtimes.
5. A VM virtualises hardware and runs its own kernel; a container shares the host kernel and isolates processes. A VM boots an operating system in seconds to minutes; a container starts a process in milliseconds. A VM's boundary is the hypervisor; a container's is the kernel's namespace and permission checks, a much larger attack surface.

</details>

---

## 2. ulimits

A **ulimit** (resource limit, or rlimit) caps what a **single process** may consume. Limits are set per process and inherited by children. Each has two values:

- the **soft** limit, which is enforced, and which the process may raise as far as…
- the **hard** limit, a ceiling that only root can raise.

```bash
ulimit -a                         # every soft limit for this shell
ulimit -Ha                        # the hard limits
```

| Flag | Limit | When it bites |
|---|---|---|
| `-n` | open file descriptors (`nofile`) | **"Too many open files"**: databases, proxies, anything with many connections |
| `-u` | processes and threads for the user (`nproc`) | "Resource temporarily unavailable" on fork; the fork-bomb defence |
| `-c` | core dump size (`core`) | 0 means no core dumps |
| `-f` | largest file the process may write (`fsize`) | "File size limit exceeded" |
| `-v` | virtual memory (`as`) | allocation failures |
| `-s` | stack size (`stack`) | deep recursion crashes |
| `-l` | locked memory (`memlock`) | databases and search engines that pin memory |
| `-t` | CPU seconds (`cpu`) | the process is killed when it has used that much CPU time |

```bash
ulimit -n                         # soft nofile for this shell
ulimit -Hn                        # hard
ulimit -Sn 2048                   # set the soft limit (up to the hard limit)
ulimit -n                         # 2048: applies to this shell and what it starts from now on
```

A limit lowered in a shell cannot be raised again above the hard limit without root, so experiments belong in a **subshell**, whose limits vanish when it exits.

```bash
( ulimit -n 20; python3 -c 'fs = [open("/dev/null") for _ in range(50)]' )
#   OSError: [Errno 24] Too many open files

( ulimit -f 100; dd if=/dev/zero of=big bs=1k count=200; ls -l big ); rm -f big
#   File size limit exceeded, and the file stops at about 100 KB

ulimit -n                         # your shell is unaffected
```

### The limits of a running process

What matters is the limit of the **process that is failing**, not of your shell.

```bash
cat /proc/$$/limits                               # this shell
P=$(pgrep -o dockerd); sudo cat /proc/$P/limits   # the Docker daemon
sudo prlimit --pid "$P"                           # the same, formatted
sudo prlimit --pid "$P" --nofile                  # one limit
# sudo prlimit --pid PID --nofile=4096:8192       # change a RUNNING process's limits (soft:hard)

sudo ls /proc/$P/fd | wc -l                       # how many descriptors it has open right now
```

Diagnosing "Too many open files": compare the open count with `Max open files` for that PID. If the count sits at the limit, either the limit is too low for the workload or the program leaks descriptors (the count only ever grows; many sockets in `CLOSE-WAIT`, Lab 15).

### Where limits come from

This is the part people get wrong. There are three separate sources, and each applies to different processes.

| Process started by | Limits come from | Configure in |
|---|---|---|
| a login session (SSH, console, `su -`) | PAM `pam_limits` | `/etc/security/limits.conf`, `/etc/security/limits.d/*.conf` |
| **systemd (every service)** | the unit | `LimitNOFILE=`, `LimitNPROC=`, `LimitCORE=`… in the unit or a drop-in; defaults in `/etc/systemd/system.conf` |
| **a container runtime** | the runtime | `docker run --ulimit`, Compose `ulimits:`, or `default-ulimits` in `/etc/docker/daemon.json` |

**`/etc/security/limits.conf` has no effect on services or containers.** Editing it and wondering why the database still reports 1024 is a classic afternoon lost.

```bash
grep -vE '^\s*(#|$)' /etc/security/limits.conf /etc/security/limits.d/*.conf 2>/dev/null
systemctl show docker -p LimitNOFILE -p LimitNPROC -p LimitCORE
systemctl show sshd   -p LimitNOFILE
systemctl show -p DefaultLimitNOFILE                # systemd's default for services
```

The `limits.conf` format, for reference:

```
# domain      type   item     value
walter        soft   nofile   8192
walter        hard   nofile   65536
@developers   -      nproc    4096          # "-" sets both; "@" is a group
*             hard   core     0
```

For a service, with a drop-in (Lab 09):

```ini
[Service]
LimitNOFILE=65536
```

### In containers

```bash
docker exec lab-sleep sh -c 'ulimit -n; ulimit -u'                       # inherited from the Docker daemon's defaults
docker run --rm --ulimit nofile=64:128 alpine:3 sh -c 'ulimit -Sn; ulimit -Hn'      # 64 and 128
docker run --rm --ulimit nofile=64:128 alpine:3 sh -c 'ulimit -n 200' || echo "cannot exceed the hard limit"
docker inspect -f '{{.HostConfig.Ulimits}}' lab-sleep
```

Databases document the limits they need. Neo4j, for example, wants a `nofile` of at least 40000 and warns at start-up if it is lower. Check any database container you run:

```bash
# docker exec YOUR_DB_CONTAINER sh -c 'ulimit -n'
# sudo grep 'open files' /proc/$(docker inspect -f '{{.State.Pid}}' YOUR_DB_CONTAINER)/limits
```

### System-wide ceilings

Above all per-process limits sit kernel-wide ones:

```bash
cat /proc/sys/fs/file-max          # maximum open files for the whole system
cat /proc/sys/fs/file-nr           # allocated, unused, maximum
cat /proc/sys/fs/nr_open           # the highest value any process's nofile may be set to
cat /proc/sys/kernel/pid_max; cat /proc/sys/kernel/threads-max
```

### ulimits versus cgroups

| | ulimit | cgroup |
|---|---|---|
| Applies to | one process (inherited by children, each counted separately) | a **group** of processes, counted together |
| Can cap total memory of a service or container | no | yes |
| Can cap CPU share | no (only total CPU seconds) | yes |
| Typical use | file descriptors, core dumps, per-user process count | memory, CPU, I/O and process limits for services and containers |

Limiting a container's memory is a cgroup job. Limiting its open files is a ulimit job.

### Exercises 2

1. What are your shell's soft and hard `nofile` limits? Where did they come from?
2. In a subshell, set `nofile` to 16 and run `ls -l /`. Does it work? Lower it until `ls` fails. What is the minimum, and why does a program need a few descriptors before it opens anything?
3. For the Docker daemon and for `sshd`: report `Max open files`, the number currently open, and where the limit is configured.
4. Start a container with a `nofile` limit of 32 and prove from the **host** (`/proc/PID/limits`) that the limit is applied.
5. A service fails under load with "Too many open files". Its limit is 1024. Write the drop-in that raises it to 65536, and the commands to apply and verify. Why would editing `limits.conf` not help?
6. Which limit protects a machine from a user's runaway script that forks endlessly? What is it set to for your account?

<details><summary>Solutions</summary>

```bash
ulimit -Sn; ulimit -Hn                                                 # 1: from pam_limits at login (limits.conf), or systemd's defaults if unset
( ulimit -n 16; ls -l / > /dev/null && echo ok )                       # 2
( ulimit -n 3; ls / ) 2>&1 | tail -1                                   #    0, 1 and 2 are stdin, stdout, stderr; the loader then needs
                                                                       #    more to open shared libraries and the directory itself
for u in docker sshd; do P=$(systemctl show $u -p MainPID --value); echo "$u: $(sudo grep 'open files' /proc/$P/limits) / open now: $(sudo ls /proc/$P/fd | wc -l)"; done   # 3
systemctl show docker sshd -p LimitNOFILE
docker run -d --name lab-ul --label lab=16 --ulimit nofile=32:32 alpine:3 sleep 600                # 4
sudo grep 'open files' /proc/$(docker inspect -f '{{.State.Pid}}' lab-ul)/limits; docker rm -f lab-ul
```

5.
```bash
sudo systemctl edit myservice        # [Service]  LimitNOFILE=65536
sudo systemctl restart myservice
systemctl show myservice -p LimitNOFILE; sudo grep 'open files' /proc/$(systemctl show myservice -p MainPID --value)/limits
```
`limits.conf` is applied by PAM to login sessions. systemd starts services without PAM, so it never reads that file.

6. `nproc` (`ulimit -u`), plus the cgroup `pids` limit on your user slice: `systemctl show user-$(id -u).slice -p TasksMax`.

</details>

---

## 3. cgroups

A **control group** is a set of processes whose combined resource use the kernel measures and can limit. Where a ulimit says "this process may open 1024 files", a cgroup says "these 40 processes together may use 512 MB of memory and half a CPU".

Rocky 9 uses **cgroups v2**: one unified tree, exposed as a filesystem at `/sys/fs/cgroup`. Directories are groups. Files in them are the interface.

```bash
stat -fc %T /sys/fs/cgroup                # cgroup2fs = v2
cat /sys/fs/cgroup/cgroup.controllers     # available controllers: cpu memory io pids cpuset ...
ls /sys/fs/cgroup/
```

**systemd owns the tree.** Every service, user session and container is placed in its own cgroup automatically, which is how `systemctl status` knows a service's memory use and every one of its processes.

```bash
systemd-cgls --no-pager | head -40        # the tree: slices, services, scopes, and the processes in each
systemd-cgtop -n 1 --order=memory | head -15      # live resource use per group
cat /proc/$$/cgroup                       # which cgroup your shell is in
cat /proc/$PID/cgroup                     # which cgroup the lab container is in
```

| Top level | Contains |
|---|---|
| `system.slice` | system services, and with Docker's systemd driver, containers (`docker-<id>.scope`) |
| `user.slice` | login sessions, one `user-UID.slice` per user |
| `machine.slice` | virtual machines and systemd-nspawn containers |
| `kubepods.slice` | pods, on a Kubernetes node |

### The interface files

```bash
CG=/sys/fs/cgroup/system.slice/sshd.service
ls $CG
cat $CG/cgroup.procs                      # PIDs in this group
cat $CG/memory.current                    # bytes in use now
cat $CG/memory.max                        # the hard limit ("max" = none)
cat $CG/pids.current $CG/pids.max
cat $CG/cpu.stat | head -4                # CPU time used, in microseconds
```

| File | Meaning |
|---|---|
| `cgroup.procs` | member PIDs |
| `memory.current` | memory in use, including page cache charged to the group |
| `memory.max` | **hard limit**. Exceeding it triggers reclaim, then the OOM killer inside the group |
| `memory.high` | throttle threshold: the group is slowed and reclaimed, not killed |
| `memory.swap.max` | swap the group may use |
| `memory.events` | counters, including `oom_kill` |
| `cpu.max` | `QUOTA PERIOD` in microseconds. `50000 100000` = half a CPU. `max` = unlimited |
| `cpu.weight` | relative share under contention (default 100) |
| `cpu.stat` | usage, plus `nr_throttled` and `throttled_usec` |
| `pids.max` / `pids.current` | process and thread count limit / current |
| `io.max` | per-device bandwidth and IOPS limits |
| `cpuset.cpus` | which CPUs the group may run on |

You rarely write these files by hand. systemd and container runtimes write them from higher-level settings.

### Limits through systemd

`systemd-run` starts a command as a transient unit with any properties you like, which makes it a quick way to run something under limits without Docker.

```bash
sudo systemd-run --unit=lab-cg --collect -p MemoryMax=50M -p MemorySwapMax=0 -p CPUQuota=20% -p TasksMax=30 sleep 600
systemctl status lab-cg --no-pager | head -12
CG=/sys/fs/cgroup/system.slice/lab-cg.service
cat $CG/memory.max                        # 52428800  = 50 * 1024 * 1024
cat $CG/cpu.max                           # 20000 100000  = 20% of one CPU
cat $CG/pids.max                          # 30
sudo systemctl stop lab-cg
```

| systemd property | cgroup file | Docker flag | Kubernetes |
|---|---|---|---|
| `MemoryMax=` | `memory.max` | `--memory` | `limits.memory` |
| `MemoryHigh=` | `memory.high` | `--memory-reservation` (approx.) | |
| `MemorySwapMax=` | `memory.swap.max` | `--memory-swap` | |
| `CPUQuota=50%` | `cpu.max` | `--cpus 0.5` | `limits.cpu` |
| `CPUWeight=` | `cpu.weight` | `--cpu-shares` | `requests.cpu` |
| `TasksMax=` | `pids.max` | `--pids-limit` | |
| `AllowedCPUs=` | `cpuset.cpus` | `--cpuset-cpus` | |
| `IOReadBandwidthMax=` | `io.max` | `--device-read-bps` | |

The same four files sit under a systemd service, a Docker container and a Kubernetes pod. Learn them once.

### Watch a memory limit being enforced

```bash
sudo systemd-run --unit=lab-oom -p MemoryMax=50M -p MemorySwapMax=0 \
  python3 -c 'data = b"x" * (200 * 1024 * 1024); import time; time.sleep(60)'
sleep 2
systemctl status lab-oom --no-pager | head -8          # failed (Result: oom-kill), status=9/KILL
sudo journalctl -u lab-oom --no-pager | tail -4        # "A process of this unit has been killed by the OOM killer."
sudo journalctl -k --since "1 minute ago" --no-pager | grep -iE 'oom|killed process' | tail -5
sudo systemctl reset-failed lab-oom
```

The process asked for 200 MB inside a 50 MB group. The kernel could not reclaim enough, so it killed the process with SIGKILL. From outside, that is exit status **137** (128 + 9) with no error message from the program itself, because it never got the chance to print one.

### Watch a CPU limit being enforced

```bash
sudo systemd-run --unit=lab-cpu -p CPUQuota=20% sh -c 'while :; do :; done'
sleep 5
top -b -n 1 -p "$(systemctl show lab-cpu -p MainPID --value)" | tail -2      # about 20% CPU, although it wants 100%
grep -E 'nr_periods|nr_throttled|throttled_usec' /sys/fs/cgroup/system.slice/lab-cpu.service/cpu.stat
sleep 5
grep -E 'nr_throttled' /sys/fs/cgroup/system.slice/lab-cpu.service/cpu.stat     # still climbing
sudo systemctl stop lab-cpu
```

A CPU limit **throttles**; it does not kill. The process is paused for the rest of each 100 ms period once its quota is spent. In an application this appears as latency spikes with low average CPU use, and `nr_throttled` rising is the evidence. That is the commonest hidden cause of "slow in the container, fast on my laptop".

### The same thing in Docker

```bash
docker run -d --name lab-limited --label lab=16 --memory 64m --memory-swap 64m --cpus 0.5 --pids-limit 50 alpine:3 sleep 86400
CID=$(docker inspect -f '{{.Id}}' lab-limited)
CG=/sys/fs/cgroup/system.slice/docker-$CID.scope
cat $CG/memory.max $CG/cpu.max $CG/pids.max            # 67108864, "50000 100000", 50
docker stats --no-stream lab-limited
docker info 2>/dev/null | grep -iE 'cgroup (driver|version)'
```

If the directory is not there, your Docker uses the `cgroupfs` driver; look under `/sys/fs/cgroup/docker/$CID/`.

Limits can be changed on a running container:

```bash
docker update --memory 128m --memory-swap 128m lab-limited && cat $CG/memory.max
```

A container OOM kill:

```bash
docker run --name lab-oomc --label lab=16 --memory 50m --memory-swap 50m alpine:3 tail /dev/zero; echo "exit status: $?"
docker inspect -f 'OOMKilled={{.State.OOMKilled}} ExitCode={{.State.ExitCode}}' lab-oomc
docker rm lab-oomc
```

`tail /dev/zero` buffers an endless line and grows until the cgroup limit kills it: exit 137, `OOMKilled=true`. When a container "randomly restarts", this is the first thing to check, followed by `journalctl -k | grep -i oom`.

A container **without** a memory limit can consume the host's memory, at which point the host's OOM killer chooses a victim from the whole machine, possibly your database. Set a memory limit on every long-running container.

Inside a container, `free` and `/proc/meminfo` report the **host's** memory, not the limit. Older runtimes (and badly configured JVMs) size themselves from that and get killed. `cat /sys/fs/cgroup/memory.max` inside the container gives the truth.

### Exercises 3

1. Which cgroup is your shell in? Which is `dockerd` in? Which is `lab-sleep` in?
2. Using only files under `/sys/fs/cgroup`, report the current memory use and process count of the Docker service and of one real container.
3. Do your real containers have memory limits? One command to list name and limit for all of them. For any without: what would you set, and why?
4. Run a busy loop under `CPUQuota=50%` and another under `CPUQuota=10%`. Compare `nr_throttled` after 10 seconds.
5. Run a transient unit with `TasksMax=10` that tries to start 30 background sleeps. What happens? Where is it logged?
6. Has anything on the tower been OOM-killed since boot?
7. A container's `memory.max` is 512 MB and `memory.current` sits near 500 MB, yet the application reports using 200 MB. What accounts for the difference, and is it a problem?

<details><summary>Solutions</summary>

```bash
# 1
cat /proc/$$/cgroup; cat /proc/$(pgrep -o dockerd)/cgroup; cat /proc/$(docker inspect -f '{{.State.Pid}}' lab-sleep)/cgroup

# 2
cat /sys/fs/cgroup/system.slice/docker.service/{memory.current,pids.current}
CID=$(docker inspect -f '{{.Id}}' YOUR_CONTAINER); cat /sys/fs/cgroup/system.slice/docker-$CID.scope/{memory.current,pids.current}

# 3
docker ps -q | xargs docker inspect -f '{{.Name}}  memory={{.HostConfig.Memory}}  cpus={{.HostConfig.NanoCpus}}'
# 0 means unlimited. Choose from observed peak use (docker stats over a busy period) plus headroom.

# 4
sudo systemd-run --unit=lab-c50 -p CPUQuota=50% sh -c 'while :; do :; done'
sudo systemd-run --unit=lab-c10 -p CPUQuota=10% sh -c 'while :; do :; done'
sleep 10; grep nr_throttled /sys/fs/cgroup/system.slice/lab-c{50,10}.service/cpu.stat
sudo systemctl stop lab-c50 lab-c10

# 5
sudo systemd-run --unit=lab-pids -p TasksMax=10 sh -c 'for i in $(seq 30); do sleep 60 & done; wait'
sleep 2; sudo journalctl -u lab-pids --no-pager | tail -3     # "fork: retry: Resource temporarily unavailable"
sudo journalctl -k --since "1 minute ago" | grep -i 'fork rejected by pids controller'
sudo systemctl stop lab-pids

# 6
sudo journalctl -k -b --no-pager | grep -ciE 'out of memory|oom-kill'
```

7. Page cache. Files the container reads and writes are cached in memory and charged to its cgroup. The kernel reclaims cache before resorting to the OOM killer, so it is not a problem by itself. `memory.stat` shows the split between `anon` (application memory) and `file` (cache).

</details>

---

## 4. Container runtime

"Docker" is several programs stacked on top of each other. Each level has alternatives, and Kubernetes plugs in below the top.

```
   you ──▶ docker CLI                         a client: talks to the daemon over /var/run/docker.sock
              │  REST API
              ▼
           dockerd  (Docker Engine)           images, builds, networks, volumes, the API
              │  gRPC
              ▼
           containerd                         HIGH-LEVEL runtime: pulls and stores images, manages container lifecycles
              │
              ▼
           containerd-shim-runc-v2            one per container: holds it open, so daemons can restart without killing it
              │
              ▼
           runc                               LOW-LEVEL runtime: creates namespaces and cgroups, execs your process, then exits
              │
              ▼
           your process                       running on the host kernel
```

| Level | Does | Examples |
|---|---|---|
| **Low-level (OCI) runtime** | given a root filesystem and a JSON spec, creates the namespaces and cgroups and starts the process | `runc` (the reference, in Go), `crun` (in C, default for Podman on RHEL), `runsc` (gVisor: a user-space kernel for stronger isolation), `kata-runtime` (each container in a small VM) |
| **High-level runtime** | pulls, unpacks and stores images; supervises containers; calls the low-level runtime | `containerd`, `CRI-O` |
| **Engine / tooling** | builds images, provides the CLI and API people use | Docker Engine, Podman, nerdctl |

### The standards that make the pieces swappable

| Standard | Defines |
|---|---|
| **OCI Image Spec** | the image format: layers, config, manifest. An image built by Docker runs under Podman, containerd or CRI-O |
| **OCI Runtime Spec** | the `config.json` handed to a low-level runtime |
| **OCI Distribution Spec** | the registry API |
| **CRI** (Container Runtime Interface) | the API **Kubernetes** uses to ask a runtime to run pods |

Kubernetes talks CRI. containerd and CRI-O implement it. Docker Engine does not, and Kubernetes dropped its built-in adapter for it in version 1.24. Images built with Docker work on any cluster regardless, because the image format is the shared standard. **k3s** ships its own containerd, so a k3s node needs no Docker at all.

**Podman** is the RHEL family's default tool: the same commands as Docker (`alias docker=podman` mostly works), no daemon, rootless by default, and it generates systemd units for containers. On a Rocky machine that did not already have Docker, it would be the natural choice.

### See the layers on the tower

```bash
docker info 2>/dev/null | grep -iE 'server version|storage driver|cgroup|runtimes|default runtime|containerd version|runc version|security options|root dir'
systemctl status docker containerd --no-pager | grep -E '●|Active|Main PID'
ps -eo pid,ppid,cmd | grep -E '[d]ockerd|[c]ontainerd' | cut -c1-140
runc --version
containerd --version
```

One shim per running container, each a child of PID 1 and not of the daemons:

```bash
ps -eo pid,ppid,cmd | grep '[c]ontainerd-shim' | cut -c1-120
pstree -sp "$(docker inspect -f '{{.State.Pid}}' lab-sleep)"
```

That parentage is deliberate. The shim keeps the container's stdio and exit status, so `containerd` and (with `live-restore`) `dockerd` can be upgraded without stopping containers.

Docker's containers, seen directly in containerd. Docker uses a containerd namespace called `moby`:

```bash
sudo ctr namespaces list
sudo ctr --namespace moby containers list | head
sudo ctr --namespace moby tasks list | head            # running tasks, with host PIDs
```

And the OCI runtime spec that was generated for `lab-sleep` and handed to `runc`:

```bash
CID=$(docker inspect -f '{{.Id}}' lab-sleep)
B=/run/containerd/io.containerd.runtime.v2.task/moby/$CID
sudo ls $B                                             # config.json, rootfs, ...
sudo dnf install -y jq >/dev/null
sudo jq '.process.args, .process.user, .hostname' $B/config.json
sudo jq '.linux.namespaces' $B/config.json             # the namespaces from section 1
sudo jq '.linux.resources' $B/config.json              # the cgroup limits from section 3
sudo jq '.process.rlimits' $B/config.json              # the ulimits from section 2
sudo jq '.process.capabilities.bounding' $B/config.json
```

That file is where the three earlier sections meet. Everything `docker run` does comes down to writing this JSON and asking `runc` to act on it.

### Exercises 4

1. Which low-level runtime, storage driver, cgroup driver and cgroup version does your Docker use?
2. How many `containerd-shim` processes are running, and does the number match `docker ps -q | wc -l`?
3. From `lab-limited`'s `config.json`, read out its memory limit, CPU quota and pids limit. Do they match what you passed to `docker run`?
4. In the chain CLI → dockerd → containerd → shim → runc, which components are still running while a container runs, and which have already exited?
5. Why can an image built with `docker build` run on a Kubernetes cluster that has no Docker installed?
6. Name one reason to choose each of: `crun`, gVisor, Kata Containers.

<details><summary>Solutions</summary>

```bash
docker info 2>/dev/null | grep -iE 'default runtime|storage driver|cgroup'                # 1: runc, overlay2, systemd, 2
ps -eo cmd | grep -c '[c]ontainerd-shim'; docker ps -q | wc -l                            # 2
CID=$(docker inspect -f '{{.Id}}' lab-limited)                                            # 3
sudo jq '.linux.resources | {memory: .memory.limit, cpu: .cpu, pids: .pids.limit}' /run/containerd/io.containerd.runtime.v2.task/moby/$CID/config.json
```

4. Running: `dockerd`, `containerd`, one shim per container, and your process. The `docker` CLI exits once it has made its request (unless attached), and `runc` exits as soon as the container's process has started.
5. The image format is an OCI standard. The cluster's runtime (containerd or CRI-O) pulls and runs OCI images whoever built them.
6. `crun`: faster start and lower memory than runc. gVisor: syscalls are handled by a user-space kernel, shrinking the host-kernel attack surface for untrusted workloads. Kata: each container gets its own lightweight VM and kernel, for hard multi-tenant isolation.

</details>

---

## 5. Docker

### Images and containers

An **image** is a read-only template: a filesystem plus metadata (default command, environment, exposed ports). A **container** is a running, or stopped, instance of one. The relationship is program to process.

An image name has four parts:

```
docker.io / library/nginx : 1.27-alpine @ sha256:…
└ registry ┘ └ repository ┘ └── tag ───┘ └ digest ┘
```

Registry defaults to Docker Hub, tag defaults to `latest`. A **tag is a movable label**; `nginx:latest` today and next month are different images. A **digest** identifies exact content. Pin versions in anything you deploy.

```bash
docker images                                          # local images
docker pull nginx:alpine
docker image inspect nginx:alpine -f '{{.Id}} {{.Architecture}} {{.Size}}'
docker image inspect nginx:alpine -f '{{json .Config.Cmd}} {{json .Config.ExposedPorts}}'
docker history nginx:alpine                            # the layers, and the instruction that made each
docker images --digests nginx
```

### Running

```bash
docker run --rm alpine:3 echo "hello"                  # run, print, exit, remove (--rm)
docker run --rm -it alpine:3 sh                        # -i keep stdin, -t allocate a terminal: an interactive shell. "exit" to leave.
docker run -d --name lab-web1 --label lab=16 -p 127.0.0.1:8088:80 nginx:alpine     # -d detached, a name, a published port
curl -sI http://127.0.0.1:8088/ | head -1
```

| Option | Meaning |
|---|---|
| `-d` | run in the background |
| `--name N` | a name you choose (otherwise a random one) |
| `--rm` | delete the container when it exits |
| `-it` | interactive terminal |
| `-p [IP:]HOST:CONTAINER` | publish a port. **Give the IP**: `127.0.0.1:` for local only (Lab 13) |
| `-e KEY=value`, `--env-file F` | environment variables |
| `-v NAME_or_PATH:/in/container[:opts]` | a volume or bind mount |
| `--network N` | attach to a network |
| `--restart unless-stopped` | restart policy |
| `--memory`, `--cpus`, `--pids-limit`, `--ulimit` | sections 2 and 3 |
| `-u UID[:GID]` | run as this user |
| `--label k=v` | metadata you can filter on |
| `-w DIR` | working directory |
| `--entrypoint CMD` | override the image's entrypoint |
| `--init` | add a small init as PID 1 to forward signals and reap zombies |

Every option after the image name is passed to the container's command: `docker run alpine:3 ls -l /etc`.

### Managing

```bash
docker ps                                              # running
docker ps -a                                           # all, including exited
docker ps -a --filter label=lab=16 --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker logs lab-web1                                   # the container's stdout and stderr
docker logs -f --tail 20 --timestamps lab-web1         # follow (Ctrl-C)
docker exec lab-web1 nginx -v                          # run a command in a running container
docker exec -it lab-web1 sh                            # a shell inside (exit to leave)
docker top lab-web1                                    # its processes
docker stats --no-stream                               # CPU, memory, network, I/O per container
docker inspect lab-web1 | less                         # everything, as JSON
docker inspect -f '{{.State.Status}} {{.NetworkSettings.IPAddress}} {{.HostConfig.RestartPolicy.Name}}' lab-web1
docker port lab-web1
docker cp lab-web1:/etc/nginx/nginx.conf ./nginx.conf.copy          # copy a file out (or in)
```

```bash
docker stop lab-web1                                   # SIGTERM, wait 10 s, then SIGKILL
docker start lab-web1
docker restart lab-web1
docker kill -s HUP lab-web1                            # send a specific signal (nginx reloads on HUP)
docker rm -f lab-web1                                  # stop and remove. CHECK THE NAME.
```

`docker inspect -f` with a Go template, or piped to `jq`, is how you script against Docker. Exit codes to recognise: 0 clean, 1 application error, 125 Docker could not run it, 126 command not executable, 127 command not found, **137 SIGKILL (OOM, or a stop that timed out)**, 143 SIGTERM.

### Building an image

A small web application with no dependencies beyond Python's standard library:

```bash
mkdir -p ~/labwork/16/app && cd ~/labwork/16/app

cat > app.py <<'EOF'
import os
import socket
import time
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

START = time.time()


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/health":
            body = "ok\n"
        else:
            body = (
                f"hello from {socket.gethostname()}\n"
                f"greeting: {os.environ.get('GREETING', '(not set)')}\n"
                f"pid {os.getpid()}, uid {os.getuid()}, up {time.time() - START:.0f}s\n"
            )
        self.send_response(200)
        self.send_header("Content-Type", "text/plain")
        self.end_headers()
        self.wfile.write(body.encode())

    def log_message(self, fmt, *args):
        print(f"{self.address_string()} {fmt % args}", flush=True)


print("listening on :8000", flush=True)
ThreadingHTTPServer(("0.0.0.0", 8000), Handler).serve_forever()
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-alpine

RUN adduser -D -u 10001 app
WORKDIR /app
COPY app.py .

USER app
EXPOSE 8000
CMD ["python", "app.py"]
EOF

cat > .dockerignore <<'EOF'
.git
__pycache__
*.pyc
.env
EOF

docker build -t lab-web:1 .
docker images lab-web
docker history lab-web:1
```

| Instruction | Does |
|---|---|
| `FROM` | the base image. Small, official, and pinned to a version |
| `RUN` | run a command at **build** time and save the result as a layer |
| `WORKDIR` | set (and create) the working directory |
| `COPY src dst` | copy files from the build context into the image |
| `ENV`, `ARG` | runtime environment variables / build-time variables |
| `USER` | the user for later instructions and for the running container |
| `EXPOSE` | documents the port. It does **not** publish it |
| `CMD` | the default command when a container starts |
| `ENTRYPOINT` | a fixed executable, with `CMD` supplying default arguments |
| `HEALTHCHECK` | a command Docker runs to judge health |
| `LABEL` | metadata |

`.dockerignore` keeps files out of the **build context**, the directory sent to the daemon. Without it, secrets in `.env` and the whole `.git` history can end up inside an image.

Run it:

```bash
docker run -d --name lab-app --label lab=16 -p 127.0.0.1:8090:8000 -e GREETING="built by hand" lab-web:1
curl -s http://127.0.0.1:8090/
docker logs lab-app
docker exec lab-app id                                 # uid=10001(app): not root
```

### Layers and the build cache

Each instruction creates a layer, and Docker reuses a cached layer when the instruction and its inputs are unchanged. **Once one layer changes, every layer after it is rebuilt.** So put what changes rarely first, and what changes often last.

```bash
docker build -t lab-web:1 .                            # every step says CACHED
echo "# a comment" >> app.py
docker build -t lab-web:1 .                            # FROM and adduser are cached; COPY and later are rebuilt
```

With dependencies, the standard pattern follows directly:

```dockerfile
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt     # rebuilt only when requirements.txt changes
COPY . .                                               # code changes do not invalidate the install layer
```

Other habits that keep images small and builds fast: chain related commands in one `RUN` and clean up in the same instruction (a file deleted in a later layer still occupies the earlier one); use a **multi-stage build** to compile in one image and copy only the result into a small final image; never bake secrets into any layer.

### PID 1 and stopping cleanly

```bash
time docker stop lab-app                               # about 10 seconds
docker inspect -f 'exit={{.State.ExitCode}}' lab-app   # 137
```

`docker stop` sent SIGTERM, nothing happened, and ten seconds later it sent SIGKILL. The reason is Lab 07 meeting section 1: the application is **PID 1** in its namespace, and the kernel does not apply default signal actions to PID 1. A signal PID 1 has no handler for is ignored. Three fixes:

```bash
docker rm lab-app
docker run -d --name lab-app --label lab=16 --init -p 127.0.0.1:8090:8000 lab-web:1     # fix 1: --init
time docker stop lab-app                               # immediate
docker rm lab-app
```

Fix 2 is to handle the signal in the application, which is the right answer for anything with connections to finish or data to flush. Add a handler and rebuild as version 2:

```bash
python3 - <<'EOF'
src = open("app.py").read()
src = src.replace("import os\n", "import os\nimport signal\nimport sys\n", 1)
handler = '''
def stop(signum, frame):
    print("SIGTERM received, shutting down", flush=True)
    sys.exit(0)


signal.signal(signal.SIGTERM, stop)
'''
src = src.replace('\nprint("listening on :8000", flush=True)', handler + 'print("listening on :8000", flush=True)', 1)
open("app.py", "w").write(src)
EOF
grep -n -A6 'def stop' app.py

docker build -t lab-web:2 .
docker run -d --name lab-app --label lab=16 -p 127.0.0.1:8090:8000 lab-web:2
time docker stop lab-app                               # immediate
docker logs lab-app | tail -1                          # SIGTERM received, shutting down
docker inspect -f 'exit={{.State.ExitCode}}' lab-app   # 0
docker rm lab-app
```

Fix 3 concerns the Dockerfile. `CMD ["python", "app.py"]` is the **exec form**: the program itself is PID 1. `CMD python app.py` is the **shell form**: `/bin/sh -c` is PID 1, your program is its child, and the shell does not pass SIGTERM on. Always use the exec form, and in an entrypoint script end with `exec "$@"`.

### Data: volumes and bind mounts

A container's writable layer dies with the container. Persistent data lives outside it.

| | Named volume | Bind mount | tmpfs |
|---|---|---|---|
| Syntax | `-v lab-data:/data` | `-v /host/path:/data` | `--tmpfs /tmp` |
| Lives in | `/var/lib/docker/volumes/…`, managed by Docker | wherever you say on the host | memory |
| Use for | databases, application state | config files, source code in development | scratch space, secrets |

```bash
docker volume create --label lab=16 lab-data
docker run --rm -v lab-data:/data alpine:3 sh -c 'echo "written $(date)" >> /data/log.txt'
docker run --rm -v lab-data:/data alpine:3 cat /data/log.txt        # a different container, the same data
docker volume inspect lab-data -f '{{.Mountpoint}}'
docker volume ls --filter label=lab=16
```

Back up a volume by mounting it into a throwaway container next to a bind mount:

```bash
mkdir -p ~/labwork/16/backup
docker run --rm -v lab-data:/data:ro -v ~/labwork/16/backup:/backup:Z alpine:3 tar -czf /backup/lab-data.tgz -C /data .
sudo tar -tzf ~/labwork/16/backup/lab-data.tgz
```

That one-liner works for any volume, and consistent backups of a database additionally need the database's own dump tool or a stopped container.

**Bind mounts on Rocky need an SELinux label.** A container process runs as `container_t` and may only touch files labelled for containers. Your home directory's files are not.

```bash
mkdir -p ~/labwork/16/site && echo "<h1>bind mounted</h1>" > ~/labwork/16/site/index.html
docker run -d --name lab-bind --label lab=16 -p 127.0.0.1:8091:80 -v ~/labwork/16/site:/usr/share/nginx/html:ro nginx:alpine
curl -s http://127.0.0.1:8091/ | head -3               # 403 Forbidden
sudo ausearch -m AVC -ts recent 2>/dev/null | grep -c container_t       # the denial, in the audit log
docker rm -f lab-bind

docker run -d --name lab-bind --label lab=16 -p 127.0.0.1:8091:80 -v ~/labwork/16/site:/usr/share/nginx/html:ro,Z nginx:alpine
curl -s http://127.0.0.1:8091/                         # works
ls -Zd ~/labwork/16/site                               # relabelled container_file_t
docker rm -f lab-bind
```

`:Z` relabels the directory for this container only; `:z` for sharing between containers. **Never use either on a system directory** such as `/etc`, `/home` or `/var`: relabelling those breaks the host. Named volumes need no option.

Where does your real data live? For any container you depend on:

```bash
# docker inspect -f '{{range .Mounts}}{{.Type}}  {{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' YOUR_CONTAINER
```

If a database's data directory does not appear there as a volume or bind mount, its data is in the container's writable layer and will be lost when the container is removed.

### Networking

| Network | Behaviour |
|---|---|
| default `bridge` | containers get addresses on `docker0`; they reach each other **by IP only** |
| **user-defined bridge** | the same, plus built-in **DNS by container name** and isolation from other networks. Use this |
| `host` | no network namespace: the container uses the host's stack directly. No port mapping, no isolation |
| `none` | loopback only |

```bash
docker network create --label lab=16 lab-net
docker run -d --name lab-api --label lab=16 --network lab-net lab-web:2
docker run --rm --network lab-net alpine:3 wget -qO- http://lab-api:8000/      # resolved by NAME
docker run --rm --network lab-net alpine:3 cat /etc/resolv.conf                # 127.0.0.11: Docker's embedded DNS
docker run --rm alpine:3 wget -qO- -T 3 http://lab-api:8000/ || echo "the default bridge cannot resolve or reach it"
docker network inspect lab-net -f '{{range .Containers}}{{.Name}} {{.IPv4Address}}{{"\n"}}{{end}}'
ip -br link show type bridge                                                   # a new br-… bridge on the host
```

`lab-api` publishes no port at all. Other containers on `lab-net` reach it on 8000; nothing else can. **Only publish what must be reached from outside the Docker network**, and on `127.0.0.1` unless it has to be reachable from other machines, because published ports bypass firewalld (Lab 13).

### Compose

Compose describes a multi-container application in one YAML file. You will put the application and a reverse proxy together, with the limits and hardening from earlier sections.

```bash
cd ~/labwork/16/app

cat > nginx.conf <<'EOF'
server {
    listen 80;
    location / {
        proxy_pass http://app:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF

cat > compose.yaml <<'EOF'
name: lab16

services:
  app:
    build: .
    image: lab-web:2
    environment:
      GREETING: "hello from compose"
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    security_opt: ["no-new-privileges:true"]
    mem_limit: 128m
    cpus: 0.5
    pids_limit: 100
    ulimits:
      nofile: { soft: 1024, hard: 2048 }
    healthcheck:
      test: ["CMD", "wget", "-qO-", "http://127.0.0.1:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 5s
    labels: { lab: "16" }

  proxy:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8089:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro,Z
    depends_on:
      app:
        condition: service_healthy
    restart: unless-stopped
    mem_limit: 64m
    labels: { lab: "16" }
EOF

docker compose config --quiet && echo "compose file is valid"
docker compose up -d
docker compose ps
curl -s http://127.0.0.1:8089/
```

| Key | Meaning |
|---|---|
| `name:` | the project name: prefixes containers, networks and volumes |
| `build:` / `image:` | build from a Dockerfile, and tag the result |
| `environment:` / `env_file:` | variables. Keep secrets in a `.env` file outside version control |
| `ports:` | published ports, with the bind address |
| `volumes:` | bind mounts and named volumes |
| `depends_on:` with `condition: service_healthy` | start order that waits for a health check, not just for the process |
| `restart:` | `no`, `on-failure`, `always`, `unless-stopped` |
| `healthcheck:` | how Docker decides the service is healthy |
| `mem_limit`, `cpus`, `pids_limit`, `ulimits` | sections 2 and 3, declaratively |
| `read_only`, `cap_drop`, `security_opt`, `user` | hardening |

Compose created a network (`lab16_default`) and both services joined it, which is why nginx can reach `http://app:8000` by service name.

```bash
docker compose logs --tail 10
docker compose logs -f app                             # follow one service (Ctrl-C)
docker compose exec app id
docker compose top
docker compose exec app sh -c 'touch /app/x' || echo "read-only root filesystem: writes refused"
docker inspect -f '{{.State.Health.Status}}' lab16-app-1
```

Scale the application. nginx resolves `app` to every replica and spreads requests across them:

```bash
docker compose up -d --scale app=3
docker compose restart proxy                           # nginx resolves upstream names when it starts
for i in 1 2 3 4 5 6; do curl -s http://127.0.0.1:8089/ | head -1; done      # the hostname changes
docker compose up -d --scale app=1
```

Each replica's hostname is its container ID. This is a small-scale version of what a Kubernetes Service does for a set of pods.

Prove that the limits you declared are the cgroup files and rlimits from sections 2 and 3:

```bash
CID=$(docker compose ps -q app | head -1)
cat /sys/fs/cgroup/system.slice/docker-$(docker inspect -f '{{.Id}}' $CID).scope/{memory.max,cpu.max,pids.max}
sudo grep 'open files' /proc/$(docker inspect -f '{{.State.Pid}}' $CID)/limits
grep -E 'CapEff|NoNewPrivs' /proc/$(docker inspect -f '{{.State.Pid}}' $CID)/status      # CapEff all zeros; NoNewPrivs 1
```

```bash
docker compose down                                    # stop and remove this project's containers and network
# docker compose down -v                               # ...and its named volumes. Only when you mean to delete the data.
```

`docker compose down` only ever touches the project in the current directory, which makes it safe beside other workloads. `-v` deletes data: treat it like `rm -rf`.

### Hardening checklist

| Practice | How |
|---|---|
| Run as a non-root user | `USER` in the Dockerfile, or `user:` in Compose |
| Drop capabilities | `cap_drop: [ALL]`, add back only what is needed |
| No privilege escalation | `security_opt: ["no-new-privileges:true"]` |
| Read-only root filesystem | `read_only: true`, with `tmpfs:` for the paths that must be writable |
| Limit resources | memory, CPU and pids on every long-running container |
| Publish narrowly | `127.0.0.1:` unless it must be reachable from the network |
| Small, pinned base images | specific tags or digests; rebuild regularly for security fixes |
| No secrets in images or in `docker inspect` | env files with mode 600, or Docker secrets |
| Never mount `/var/run/docker.sock` into a container | it is root on the host |
| Keep SELinux enforcing | label bind mounts with `:Z`; do not use `--privileged` or `--security-opt label=disable` |
| Scan images | `docker scout cves IMAGE`, or Trivy |

Audit a real container against it:

```bash
# docker inspect -f 'user={{.Config.User}} privileged={{.HostConfig.Privileged}} readonly={{.HostConfig.ReadonlyRootfs}} capdrop={{.HostConfig.CapDrop}} mem={{.HostConfig.Memory}} restart={{.HostConfig.RestartPolicy.Name}}' YOUR_CONTAINER
# docker inspect -f '{{range $p, $b := .NetworkSettings.Ports}}{{$p}} -> {{$b}}{{"\n"}}{{end}}' YOUR_CONTAINER
```

### Housekeeping without collateral damage

```bash
docker system df                                       # space used by images, containers, volumes, build cache
docker system df -v | head -40
sudo sh -c 'du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -h | tail -5'     # container log files
docker inspect -f '{{.HostConfig.LogConfig}}' lab-sleep
```

With the default `json-file` log driver and no size cap, a chatty container's log grows until the disk is full. The fix is a cap in `/etc/docker/daemon.json` (applies to containers created afterwards, and needs a daemon restart, so plan it), or per service in Compose:

```yaml
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
```

Safe clean-up commands, and the ones to keep away from:

| Command | Removes | On this machine |
|---|---|---|
| `docker image prune` | dangling images only (untagged leftovers of rebuilds) | safe |
| `docker builder prune` | build cache | safe |
| `docker rm NAME`, `docker rmi IMAGE`, `docker volume rm NAME` | exactly what you name | safe if you read the name |
| `docker ps -aq --filter label=lab=16 \| xargs -r docker rm -f` | everything carrying your label | safe: this is why the lab labels things |
| `docker container prune` | **every** stopped container | a stopped database container goes too |
| `docker image prune -a` | every image not used by a container | forces re-pulls; loses images you cannot rebuild |
| `docker volume prune` | volumes not attached to any container | **data loss** if a container was removed first |
| `docker system prune -a --volumes` | all of the above | never here |

### Exercises 5

1. Run `nginx:alpine` as `lab-ex1` on `127.0.0.1:8092` with a memory limit of 32 MB and the restart policy `on-failure`. Verify each of the three settings with `docker inspect -f`.
2. Kill its main process from the **host** with `sudo kill -9 <host pid>`. What does Docker do? What does `docker inspect -f '{{.RestartCount}}'` show? Repeat with `docker stop`. Explain the difference.
3. Change the Dockerfile's `CMD` to the shell form, build as `lab-web:shell`, and compare `docker top` and `time docker stop` with version 2. Explain.
4. Add a `HEALTHCHECK` instruction to the Dockerfile so that the image carries its own health check. Build, run, and watch the status go from `starting` to `healthy`.
5. Give the Compose `app` service a named volume mounted at `/data`. With `read_only: true` still set, show that `/data` is writable and `/app` is not. Bring the project down and up again and show the data survived. Then remove it properly.
6. Start two containers on `lab-net` and one on a second network `lab-net2`. Show which pairs can resolve and reach each other. Connect the third to `lab-net` with `docker network connect` and test again.
7. Write a `docker run` command for `lab-web:2` that applies the whole hardening checklist. Confirm with `docker inspect` and by trying to write to `/` inside it.
8. How much disk does Docker use on the tower, and how much is reclaimable **without** touching volumes or tagged images?
9. Translate this to Compose: `docker run -d --name cache -p 127.0.0.1:6379:6379 -v cache-data:/data --memory 256m --restart unless-stopped redis:7-alpine`.

<details><summary>Solutions</summary>

```bash
# 1
docker run -d --name lab-ex1 --label lab=16 -p 127.0.0.1:8092:80 --memory 32m --restart on-failure nginx:alpine
docker inspect -f '{{.HostConfig.Memory}} {{.HostConfig.RestartPolicy.Name}} {{json .NetworkSettings.Ports}}' lab-ex1

# 2
sudo kill -9 "$(docker inspect -f '{{.State.Pid}}' lab-ex1)"; sleep 2
docker ps --filter name=lab-ex1 --format '{{.Status}}'; docker inspect -f '{{.RestartCount}}' lab-ex1     # restarted, count 1
docker stop lab-ex1; sleep 2; docker ps -a --filter name=lab-ex1 --format '{{.Status}}'                    # Exited, not restarted
# Exit by SIGKILL is a failure (137), so on-failure restarts it. docker stop records that you asked for the stop.
docker rm lab-ex1

# 3
sed 's/^CMD .*/CMD python app.py/' Dockerfile > Dockerfile.shell
docker build -t lab-web:shell -f Dockerfile.shell .
docker run -d --name lab-sh --label lab=16 lab-web:shell; docker top lab-sh      # /bin/sh -c "python app.py" is PID 1; python is its child
time docker stop lab-sh; docker rm lab-sh                                        # 10 s: sh ignores SIGTERM as PID 1 and does not forward it
rm Dockerfile.shell; docker rmi lab-web:shell

# 4   add before CMD:
#   HEALTHCHECK --interval=5s --timeout=2s --retries=3 CMD wget -qO- http://127.0.0.1:8000/health || exit 1
docker build -t lab-web:hc . && docker run -d --name lab-hc --label lab=16 lab-web:hc
for i in 1 2 3 4; do docker inspect -f '{{.State.Health.Status}}' lab-hc; sleep 3; done; docker rm -f lab-hc; docker rmi lab-web:hc

# 5   in compose.yaml, under app:        volumes: [ "appdata:/data" ]
#     and at the top level:              volumes: { appdata: { labels: { lab: "16" } } }
docker compose up -d
docker compose exec app sh -c 'date >> /data/stamp && cat /data/stamp; touch /app/x' 
docker compose down; docker compose up -d; docker compose exec app cat /data/stamp     # still there
docker compose down -v                                                                 # removes lab16_appdata too

# 6
docker network create --label lab=16 lab-net2
docker run -d --name lab-a --label lab=16 --network lab-net  alpine:3 sleep 600
docker run -d --name lab-b --label lab=16 --network lab-net  alpine:3 sleep 600
docker run -d --name lab-c --label lab=16 --network lab-net2 alpine:3 sleep 600
docker exec lab-a ping -c1 -W1 lab-b && echo "a -> b ok"
docker exec lab-a ping -c1 -W1 lab-c || echo "a -> c: cannot resolve, different network"
docker network connect lab-net lab-c; docker exec lab-a ping -c1 -W1 lab-c && echo "a -> c ok after connect"
docker rm -f lab-a lab-b lab-c; docker network rm lab-net2

# 7
docker run -d --name lab-hard --label lab=16 --read-only --tmpfs /tmp --cap-drop ALL --security-opt no-new-privileges \
  --memory 128m --cpus 0.5 --pids-limit 100 -u 10001:10001 -p 127.0.0.1:8093:8000 --restart unless-stopped lab-web:2
docker exec lab-hard sh -c 'touch /x' ; docker exec lab-hard id; docker rm -f lab-hard

# 8
docker system df          # reclaimable without risk: dangling images and build cache
docker image prune --filter dangling=true      # answer n unless you want to do it
```

```yaml
# 9
services:
  cache:
    image: redis:7-alpine
    container_name: cache
    ports: ["127.0.0.1:6379:6379"]
    volumes: ["cache-data:/data"]
    mem_limit: 256m
    restart: unless-stopped
volumes:
  cache-data:
```

</details>

---

## Challenge: from a process to a container, and back

1. Run `lab-web:2` as `lab-final` with: a 100 MB memory limit, half a CPU, 64 pids, `nofile` 256, read-only root, all capabilities dropped, on `lab-net`, published on `127.0.0.1:8094`.
2. Without using `docker inspect` for the answers, and using only the host's `/proc`, `/sys/fs/cgroup`, `lsns` and `ss`, produce evidence for **each** of those settings: the cgroup files, `/proc/PID/limits`, `/proc/PID/status`, `/proc/PID/mounts`, the network namespace's sockets.
3. Find its OCI `config.json` and point to the line that carries each setting.
4. From the host, capture one HTTP request to it with `tcpdump` on the right bridge interface.
5. Explain in five sentences what happened between pressing Enter on `docker run` and the Python process answering a request, naming every program involved.

<details><summary>Evidence commands</summary>

```bash
docker run -d --name lab-final --label lab=16 --network lab-net -p 127.0.0.1:8094:8000 \
  --memory 100m --memory-swap 100m --cpus 0.5 --pids-limit 64 --ulimit nofile=256:256 --read-only --cap-drop ALL lab-web:2
P=$(pgrep -n -f 'python app.py'); CG=/sys/fs/cgroup$(cut -d: -f3 /proc/$P/cgroup)
cat $CG/memory.max $CG/cpu.max $CG/pids.max                         # 104857600, 50000 100000, 64
sudo grep 'open files' /proc/$P/limits                              # 256
sudo grep -E 'CapEff|Uid' /proc/$P/status                           # CapEff 0000000000000000, Uid 10001
sudo awk '$2 == "/" {print $1, $4}' /proc/$P/mounts | cut -c1-60    # overlay ro,...
sudo lsns -p "$P"; sudo nsenter -t "$P" -n ss -tln                  # its own net namespace; listening on :8000
C=$(basename "$CG" | sed 's/docker-//; s/\.scope//')
sudo jq '{rlimits: .process.rlimits, caps: .process.capabilities.effective, ro: .root.readonly, res: .linux.resources}' \
  /run/containerd/io.containerd.runtime.v2.task/moby/$C/config.json | head -40
BR=br-$(docker network inspect lab-net -f '{{.Id}}' | cut -c1-12)
sudo tcpdump -i "$BR" -nn -A -c 8 port 8000 & sleep 1; curl -s http://127.0.0.1:8094/ > /dev/null; wait
docker rm -f lab-final
```

5. The `docker` CLI sent the request to `dockerd` over its socket. `dockerd` checked that the image was present, created the container's writable layer, set up its network (a veth pair on the `lab-net` bridge, an address, a DNAT rule for the published port) and asked `containerd` to start it. `containerd` started a `containerd-shim`, which called `runc` with a generated `config.json`. `runc` created the namespaces and the cgroup, applied the limits, capabilities, seccomp filter and SELinux label, pivoted into the image's root filesystem and exec'd `python app.py` as PID 1, then exited. The request to `127.0.0.1:8094` was forwarded by Docker's proxy and NAT rules to the container's address on the bridge, where the Python process, an ordinary host process in its own namespaces, answered it.

</details>

## Where this goes next

Every Kubernetes concept you will meet is one of the pieces from this lab under a new name. A **pod** is a group of containers sharing a network namespace. **Requests and limits** are `cpu.weight`, `cpu.max` and `memory.max`. An **OOMKilled** pod is section 3. **Liveness and readiness probes** are health checks. A **Service** is the name-based load balancing you saw with `--scale`. The **container runtime** on each node is containerd, without Docker above it. A `securityContext` is the hardening checklist.

## Check yourself

<details><summary>What is a container, in kernel terms?</summary>
A normal process (or group of processes) started in its own set of namespaces, placed in a cgroup, with an image mounted as its root filesystem, and with reduced privileges (capabilities, seccomp, SELinux). It runs on the host's kernel.
</details>

<details><summary>What do namespaces do, and what do cgroups do?</summary>
Namespaces restrict what a process can see: processes, network, mounts, hostname. Cgroups restrict and account for what a group of processes can use: memory, CPU, process count, I/O.
</details>

<details><summary>A service keeps failing with "Too many open files". You raised the limit in <code>/etc/security/limits.conf</code> and nothing changed. Why?</summary>
That file is applied by PAM to login sessions. systemd services take their limits from the unit (<code>LimitNOFILE=</code>) and containers from the runtime (<code>--ulimit</code>).
</details>

<details><summary>What is the difference between a soft and a hard ulimit?</summary>
The soft limit is the one enforced. A process may raise its own soft limit up to the hard limit. Only root can raise a hard limit.
</details>

<details><summary>A container exits with code 137. What are the two usual causes and how do you tell them apart?</summary>
Killed by the OOM killer for exceeding its memory limit, or killed by <code>docker stop</code> after ignoring SIGTERM for the grace period. <code>docker inspect -f '{{.State.OOMKilled}}'</code> and the kernel log settle it.
</details>

<details><summary>What happens when a container reaches its CPU limit? Its memory limit?</summary>
CPU: it is throttled, paused until the next period, which shows up as latency and a rising <code>nr_throttled</code>. Memory: the kernel reclaims what it can and then kills a process in the group.
</details>

<details><summary>What does each of dockerd, containerd and runc do?</summary>
<code>dockerd</code> provides the API, builds, networks and volumes. <code>containerd</code> manages images and container lifecycles. <code>runc</code> performs the low-level creation of namespaces and cgroups and starts the process, then exits.
</details>

<details><summary>Why does Kubernetes not need Docker on its nodes?</summary>
It talks to a runtime through CRI, which containerd and CRI-O implement directly. Images are in the OCI format whoever built them.
</details>

<details><summary>Why does <code>docker stop</code> take ten seconds for some containers?</summary>
The main process is PID 1 in its namespace and has no SIGTERM handler, so the signal is ignored, or a shell is PID 1 and does not forward it. Docker waits out the grace period and sends SIGKILL. Fix with a handler, the exec form of <code>CMD</code>, or <code>--init</code>.
</details>

<details><summary>What is the difference between a named volume and a bind mount, and what extra step do bind mounts need on Rocky?</summary>
A named volume is storage managed by Docker under its own directory. A bind mount exposes a host path you choose. On Rocky, SELinux requires the bind-mounted path to be labelled for containers, done with the <code>:Z</code> or <code>:z</code> mount option.
</details>

<details><summary>Why can two containers on a user-defined network reach each other by name, but not on the default bridge?</summary>
User-defined networks get Docker's embedded DNS server (127.0.0.11), which resolves container and service names. The default bridge has no such resolution.
</details>

<details><summary>Why is <code>docker volume prune</code> dangerous on a host with a database container?</summary>
It deletes every volume not attached to a container. If the database container has been removed, even briefly during a recreate, its data volume counts as unused.
</details>

## Cleanup

Everything is removed by label or by exact name. Read the first command's output before running the rest.

```bash
docker ps -a --filter label=lab=16 --format '{{.Names}}'                 # the containers about to be removed
cd ~/labwork/16/app 2>/dev/null && docker compose down -v 2>/dev/null; cd ~

docker ps -aq --filter label=lab=16 | xargs -r docker rm -f
docker volume ls -q --filter label=lab=16 | xargs -r docker volume rm
docker network ls -q --filter label=lab=16 | xargs -r docker network rm
docker rmi lab-web:1 lab-web:2 2>/dev/null
sudo systemctl stop 'lab-*' 2>/dev/null; sudo systemctl reset-failed 2>/dev/null

# confirm
docker ps -a --filter label=lab=16 -q | wc -l                            # 0
docker volume ls -q --filter label=lab=16 | wc -l                        # 0
docker network ls --filter label=lab=16 -q | wc -l                       # 0
docker images 'lab-*'                                                    # empty
docker ps --format 'table {{.Names}}\t{{.Status}}'                       # your real containers, as before

sudo rm -rf ~/labwork/16       # sudo: the relabelled and container-written files may not be yours
```

The base images (`alpine:3`, `nginx:alpine`, `python:3.12-alpine`, `koalaman/shellcheck`) are left in place. Remove any you do not want with `docker rmi NAME`, after checking with `docker ps -a --filter ancestor=NAME` that nothing of yours uses them.
