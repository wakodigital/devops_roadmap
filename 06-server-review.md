# Lab 06 — Server Review

**Roadmap topics:** Uptime and Load · Authentication Logs · Services Running · Available Memory / Disk
**Time:** about 2 hours
**Touches system state:** read-only, apart from optionally installing `sysstat` and saving one script to `/usr/local/bin`.

## What you will be able to do

- Log in to an unfamiliar server and, within a couple of minutes, say whether it is healthy, what it runs, and who has been on it.
- Read a load average correctly.
- Read `free` correctly (most people do not).
- Find what is filling a disk.

A server review is the routine you run when you first log in to a machine, when someone says "it's slow", and before and after any change. This lab is run against your real tower, which makes it more interesting than a blank VM: it has real services and real history.

## Setup

```bash
mkdir -p ~/labwork/06 && cd ~/labwork/06
sudo dnf install -y sysstat procps-ng        # sysstat gives mpstat, iostat, pidstat, sar
```

---

## 1. Uptime and load

```bash
uptime
```

```
 20:41:07 up 12 days,  3:02,  2 users,  load average: 0.42, 0.31, 0.28
```

| Part | Meaning |
|---|---|
| `up 12 days, 3:02` | time since boot. A surprise here means an unplanned reboot. |
| `2 users` | logged-in sessions |
| `load average` | averaged over the last **1**, **5** and **15** minutes |

### What load average measures

On Linux, load is the average number of tasks that are either **running or waiting for a CPU**, or **blocked in uninterruptible sleep**, which in practice means waiting on disk I/O. It is not a CPU percentage.

Two things follow:

1. **Load only means something relative to the CPU count.** A load of 6 is idle on a 16-thread machine and heavily overloaded on a 2-core one.
2. **High load is not always CPU.** A machine with idle CPUs and a stuck disk or NFS mount can show a very high load.

```bash
nproc                         # logical CPUs
lscpu | grep -E 'Model name|^CPU\(s\)|Thread|Core'
cat /proc/loadavg             # 1min 5min 15min running/total last-pid
```

Rules of thumb, with N = `nproc`:

| Load vs N | Reading |
|---|---|
| well below N | idle capacity |
| around N | fully used, no queue |
| above N | tasks are queueing; the excess is the queue length |

Read the three numbers as a trend: `8.0, 2.0, 0.5` means it just started; `0.5, 2.0, 8.0` means it is recovering.

### Finding out which kind of load it is

```bash
top                           # press 1 to show each CPU; q to quit
```

The CPU line in `top`:

```
%Cpu(s):  3.1 us,  1.0 sy,  0.0 ni, 95.6 id,  0.2 wa,  0.0 hi,  0.1 si,  0.0 st
```

| Field | Meaning | High value suggests |
|---|---|---|
| `us` | user-space code | an application is busy |
| `sy` | kernel code | heavy syscalls, networking, or context switching |
| `ni` | niced (low-priority) user code | background jobs |
| `id` | idle | |
| `wa` | idle while waiting for I/O | **slow or saturated disk** |
| `hi` / `si` | hardware / software interrupts | very high network or device activity |
| `st` | stolen by the hypervisor | a noisy neighbour (VMs only) |

```bash
vmstat 1 5                    # one line per second, 5 lines. Ignore the first: it is the average since boot.
```

| Column | Meaning |
|---|---|
| `r` | tasks waiting to run. Persistently above `nproc` = CPU saturation |
| `b` | tasks blocked on I/O |
| `si` / `so` | memory swapped in / out per second. Persistently non-zero = short of RAM |
| `bi` / `bo` | blocks read from / written to disk |
| `us sy id wa st` | as in top |

```bash
mpstat -P ALL 1 3             # per-CPU: one core pinned at 100% while the rest idle = a single-threaded bottleneck
iostat -xz 1 3                # per-disk: %util near 100 and a growing queue (aqu-sz) = saturated disk
pidstat 1 3                   # per-process CPU
cat /proc/pressure/cpu /proc/pressure/io /proc/pressure/memory
```

The last command shows **PSI** (pressure stall information): the percentage of time tasks were stalled waiting for each resource. `some avg10=0.00` means nothing was waiting over the last 10 seconds. It answers "is anything actually being delayed?" more directly than load average does. If `/proc/pressure` does not exist, PSI is switched off in your kernel; it is enabled with the boot parameter `psi=1` (Lab 12 explains kernel parameters).

### Make some load and watch it

Open a second SSH session for this.

```bash
# Terminal 1: one busy loop per CPU, for 60 seconds
for i in $(seq $(nproc)); do timeout 60 sh -c 'while :; do :; done' & done

# Terminal 2
watch -n 2 uptime             # the 1-minute figure climbs toward nproc
top                           # us near 100, the sh processes at the top
```

The loops end by themselves after 60 seconds. Watch how the 1-minute average decays afterwards while the 15-minute one barely moved.

### Exercises 1

1. What is your tower's load relative to its CPU count right now? Is it busy?
2. From `vmstat 1 5`, is the machine swapping?
3. A machine has `nproc` = 4 and a load of 12, with `top` showing `95 id, 0.5 us, 0 wa`. What is odd about this, and where would you look next?
4. Start the load generator with half as many loops as CPUs. Predict the steady-state load, then check.

<details><summary>Solutions</summary>

1. Compare `cut -d' ' -f1 /proc/loadavg` with `nproc`. A home server will usually be far below.
2. Look at `si` and `so`. Zeros mean no swapping.
3. Idle CPUs and no I/O wait with a high load means many tasks stuck in uninterruptible sleep without doing I/O: typically a hung NFS or network filesystem, or a driver problem. `ps -eo state,pid,cmd | grep '^D'` lists tasks in the D state; `dmesg -T | tail` often explains why.
4. About `nproc / 2`. Load counts runnable tasks, and you created that many.

</details>

---

## 2. Authentication logs

Who has logged in, who tried and failed, and who used sudo.

### Who is on now, and who was

```bash
who                           # current sessions
w                             # the same, plus what each is running and load averages
last -n 10                    # login history, from /var/log/wtmp
last -n 10 -i                 # show IP addresses, not hostnames
last reboot | head -5         # boot history
last -x | grep -E 'shutdown|crash' | head
sudo lastb -n 10              # FAILED logins, from /var/log/btmp
lastlog | grep -v 'Never'     # the most recent login of each account
```

`last` showing `still logged in` for a session you do not recognise, or `lastlog` showing a login for a service account that should never log in, are both things to chase immediately.

### The auth log

On Rocky and other RHEL-family systems, authentication events go to **`/var/log/secure`**. (On Debian and Ubuntu the same information is in `/var/log/auth.log`.) The journal holds it too.

```bash
sudo tail -20 /var/log/secure
sudo journalctl -u sshd --since "24 hours ago" --no-pager | tail -20
sudo journalctl _COMM=sshd -p warning --since today
```

Lines you will see:

| Line contains | Meaning |
|---|---|
| `Accepted publickey for walter from 10.0.0.5` | successful key login |
| `Accepted password for …` | successful password login (should not appear if passwords are disabled) |
| `Failed password for root from …` | wrong password for an existing account |
| `Failed password for invalid user admin from …` | attempt on an account that does not exist |
| `Invalid user … from …` | the same, logged earlier in the exchange |
| `Connection closed by authenticating user …` | client gave up before authenticating |
| `session opened for user root(uid=0) by walter(uid=1000)` | privilege change via sudo or su |
| `sudo: walter : TTY=pts/0 ; PWD=… ; USER=root ; COMMAND=…` | a sudo command, in full |
| `pam_unix(sudo:auth): authentication failure` | wrong sudo password |

### Questions to ask it

These reuse Lab 05 directly.

```bash
# Successful logins: who and from where
sudo grep 'Accepted' /var/log/secure | awk '{print $9, $11}' | sort | uniq -c | sort -rn

# Failed attempts by source address
sudo grep 'Failed password' /var/log/secure | grep -oE 'from [0-9a-f.:]+' | sort | uniq -c | sort -rn | head

# Usernames being guessed
sudo grep -oE 'invalid user [^ ]+' /var/log/secure | sort | uniq -c | sort -rn | head

# Every sudo command today
sudo journalctl _COMM=sudo --since today --no-pager | grep COMMAND

# Rotated logs too
sudo ls -l /var/log/secure*
sudo zgrep -h 'Accepted' /var/log/secure* | wc -l
```

Your tower is reached through a Cloudflare Tunnel, so its SSH port is probably not open to the internet, and the failed-password list may be empty. A machine with port 22 exposed directly collects thousands of these a day. Either way, the thing to confirm is that **every `Accepted` line is you**.

### Lockouts and the audit log

```bash
sudo faillock                          # accounts with recent failed attempts (if faillock is configured)
sudo faillock --user "$USER"           # one account
sudo aureport --auth --summary 2>/dev/null | head -20     # audit daemon's summary of authentication events
sudo aureport --login --failed 2>/dev/null | tail
```

### Exercises 2

1. When did the tower last boot, and does that match `uptime`?
2. From which IP addresses have you logged in during the period `/var/log/secure` covers?
3. Were any logins made with a password, as opposed to a key?
4. Deliberately fail a sudo password (`sudo -k; sudo true` and type nonsense once). Find the resulting log entry.
5. How far back does `last` go? (Look at the final line of its output.)

<details><summary>Solutions</summary>

```bash
# 1
who -b; uptime -s; last reboot | head -2

# 2
sudo grep 'Accepted' /var/log/secure | awk '{print $11}' | sort | uniq -c

# 3
sudo grep -c 'Accepted password' /var/log/secure

# 4
sudo journalctl _COMM=sudo -n 10 --no-pager
# look for: pam_unix(sudo:auth): authentication failure; ... user=walter
# If sudo never prompts, you have a NOPASSWD rule and there is nothing to fail.

# 5
last | tail -1          # "wtmp begins <date>"
```

</details>

---

## 3. Services running

What is this machine doing, and what is it listening for?

### systemd's view

```bash
systemctl list-units --type=service --state=running
systemctl list-units --type=service --state=running --no-legend | wc -l
systemctl --failed                                   # anything here needs attention
systemctl list-unit-files --type=service --state=enabled      # what starts at boot
systemctl list-timers                                # scheduled jobs (systemd's cron)
systemctl is-system-running                          # "running" = all good, "degraded" = something failed
```

Running and enabled are independent. **Running** is now. **Enabled** is at next boot. A service that is running but not enabled disappears after a reboot, which is a classic source of "it worked until the power cut". Check for mismatches:

```bash
comm -23 \
  <(systemctl list-units --type=service --state=running --no-legend | awk '{print $1}' | sort) \
  <(systemctl list-unit-files --type=service --state=enabled --no-legend | awk '{print $1}' | sort)
```

That lists running services that are not enabled. Many are legitimately started on demand by something else (`static` units, socket activation, `user@1000.service`). A service **you** installed appearing in that list deserves a second look.

### The network's view: what is listening

```bash
sudo ss -tulpn
```

| Flag | Meaning |
|---|---|
| `-t` / `-u` | TCP / UDP |
| `-l` | listening sockets only |
| `-p` | show the owning process (needs sudo to see other users' processes) |
| `-n` | numeric: do not resolve port or host names |

The **Local Address** column is the security-relevant one:

| Address | Reachable from |
|---|---|
| `127.0.0.1:7687` or `[::1]:…` | this machine only |
| `0.0.0.0:22` or `*:22` or `[::]:22` | every network interface |
| `192.168.1.50:8080` | that one interface |

For every line bound to `0.0.0.0`, you should be able to say what it is and why it needs to be reachable. Anything that only needs to be reached through a local reverse proxy or the Cloudflare tunnel should be bound to `127.0.0.1`.

Then check what the firewall actually lets in (details in Lab 13):

```bash
sudo firewall-cmd --list-all
```

### The process view

```bash
ps -eo pid,user,%cpu,%mem,etime,cmd --sort=-%mem | head -15
systemd-cgtop -n 1 --order=memory | head -15      # resource use grouped by service
```

### Containers are services too

systemd sees Docker as one service. What runs inside it is a second inventory.

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker ps -a --filter status=exited               # stopped containers
docker stats --no-stream
```

Compare the `Ports` column with the `ss` output: each published port shows up in `ss` as a `docker-proxy` process or a listening socket on the host.

### Exercises 3

1. How many services are running? Are any failed?
2. List every TCP port your tower listens on that is reachable from other machines, with the process behind each.
3. Pick one listening port you do not immediately recognise. Trace it: port → PID → command → systemd unit or container.
4. Is every service you depend on (Docker, the tunnel, sshd) **enabled** as well as running?
5. Which service uses the most memory?

<details><summary>Solutions</summary>

```bash
# 1
systemctl list-units --type=service --state=running --no-legend | wc -l
systemctl --failed

# 2
sudo ss -tlpn | awk 'NR == 1 || ($4 !~ /^127\./ && $4 !~ /^\[::1\]/)'

# 3
sudo ss -tlpn | grep ':7474'                 # example port; note pid=NNNN in the last column
ps -o pid,ppid,user,cmd -p NNNN
systemctl status NNNN                        # systemctl status accepts a PID and shows the owning unit
docker ps --filter publish=7474              # if the process is docker-proxy

# 4
systemctl is-enabled docker sshd cloudflared
systemctl is-active  docker sshd cloudflared

# 5
systemd-cgtop -n 1 --order=memory | head -5
```

</details>

---

## 4. Available memory and disk

### Memory

```bash
free -h
```

```
               total        used        free      shared  buff/cache   available
Mem:            62Gi       6.1Gi        41Gi       310Mi        15Gi        55Gi
Swap:          8.0Gi          0B       8.0Gi
```

| Column | Meaning |
|---|---|
| `total` | installed RAM, minus what the kernel reserves |
| `used` | memory held by processes |
| `free` | memory doing nothing at all |
| `buff/cache` | the page cache: disk contents kept in RAM to make I/O fast |
| **`available`** | an estimate of how much new applications could get without swapping: `free` plus the reclaimable part of cache |

**Read `available`, not `free`.** Linux deliberately fills otherwise-idle RAM with cache, and drops that cache instantly when a program needs the memory. A server with `free` near zero and a large `available` is healthy. "Unused RAM is wasted RAM."

Signs of a real memory shortage:

- `available` is a small fraction of `total`
- swap `used` is large **and** `vmstat` shows ongoing `si`/`so`
- the OOM killer has fired: `sudo dmesg -T | grep -i 'out of memory'` or `sudo journalctl -k | grep -i oom`

```bash
grep -E 'MemTotal|MemAvailable|SwapTotal|SwapFree|^Cached|Dirty' /proc/meminfo
ps -eo pid,user,rss,cmd --sort=-rss | head -10 | awk '{printf "%-8s %-10s %8.1f MB  %s\n", $1, $2, $3/1024, $4}'
swapon --show
sudo dmesg -T | grep -iE 'oom|out of memory' | tail
```

`RSS` (resident set size) is the physical memory a process has in RAM now. `VSZ` is the address space it has reserved, which is routinely many times larger and not a cause for concern.

### Disk space

```bash
df -h                               # every mounted filesystem
df -hT -x tmpfs -x devtmpfs -x overlay      # real disks only, with filesystem type
df -h /var/lib/docker               # the filesystem holding a particular path
```

A full filesystem is one of the most common causes of outages: databases stop accepting writes, services fail to start, and logins can fail. Watch `Use%` on `/` and on whatever holds `/var`.

Finding what is using the space, top-down:

```bash
sudo du -xh --max-depth=1 / 2>/dev/null | sort -h | tail -12
sudo du -xh --max-depth=1 /var 2>/dev/null | sort -h | tail
sudo du -xh --max-depth=1 /var/lib 2>/dev/null | sort -h | tail
```

`-x` stays on one filesystem, so `/proc` and other mounts are not counted. Repeat one level deeper each time until you reach the culprit.

```bash
sudo find / -xdev -type f -size +500M -exec ls -lh {} + 2>/dev/null     # individual big files
sudo journalctl --disk-usage                                            # the journal
docker system df                                                        # images, containers, volumes, build cache
sudo du -sh /var/cache/dnf                                              # package cache (sudo dnf clean all empties it)
```

The usual suspects on a machine like yours: `/var/lib/docker` (old images and build cache), `/var/log` and the journal, and database data directories.

### Inodes: the other way to be full

A filesystem also has a fixed number of inodes on ext4, and a dynamic but limited number on XFS. Running out gives `No space left on device` while `df -h` shows free space.

```bash
df -i
```

Millions of tiny files (session files, a mail queue, a runaway cache) cause this. Lab 11 reproduces it.

### Deleted but still open

```bash
sudo lsof +L1 2>/dev/null | head          # files with zero links that a process still holds open
```

If `df` says a filesystem is full and `du` cannot find the data, this is why: a log was deleted while the daemon still had it open. The space returns when the process closes the file or is restarted.

### Disk health

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
sudo dmesg -T | grep -iE 'i/o error|ata[0-9]+.*(error|fail)|nvme.*(error|timeout)' | tail
sudo dnf install -y smartmontools && sudo smartctl -H /dev/nvme0n1        # use your device name from lsblk
```

For an old tower, a SMART check belongs in the review.

### Exercises 4

1. How much memory is available on the tower, as a percentage of total? Compute it with awk from `/proc/meminfo`.
2. Which three processes use the most physical memory?
3. Which filesystem is fullest? What is the largest directory on it, three levels down?
4. How much disk is Docker using, and how much of that is reclaimable?
5. How much space does the journal use? What is the configured limit? (`man journald.conf`, look for `SystemMaxUse`.)
6. Is any filesystem above 50% of its inodes?

<details><summary>Solutions</summary>

```bash
# 1
awk '/MemTotal/ {t=$2} /MemAvailable/ {a=$2} END {printf "%.0f%% available\n", a / t * 100}' /proc/meminfo

# 2
ps -eo rss,pid,cmd --sort=-rss | head -4

# 3
df -h -x tmpfs -x devtmpfs -x overlay | sort -k5 -n | tail -1
sudo du -xh --max-depth=3 / 2>/dev/null | sort -h | tail -15

# 4
docker system df           # the RECLAIMABLE column

# 5
sudo journalctl --disk-usage
grep -vE '^\s*(#|$)' /etc/systemd/journald.conf
# With nothing set, the default cap is 10% of the filesystem, to a maximum of 4 GB.

# 6
df -i | awk 'NR > 1 && $5+0 > 50'
```

</details>

---

## Challenge: write your review script

Turn the review into one script that prints a compact report. This is the first piece of real tooling in the workbook; Lab 09 will schedule it with a systemd timer and Lab 14 will tidy up the scripting.

Requirements:

- hostname, OS, kernel, uptime
- load averages next to the CPU count
- memory available as a percentage
- filesystems over 80% full (or a line saying none)
- failed systemd units (or a line saying none)
- listening TCP ports that are not bound to localhost
- the last five successful SSH logins

Write it yourself before looking.

<details><summary>One solution</summary>

```bash
cat > server-review.sh <<'EOF'
#!/usr/bin/env bash
# Quick health review. Run with sudo for complete output.

section() { printf '\n== %s ==\n' "$1"; }

section "System"
. /etc/os-release
echo "Host:    $(hostname)"
echo "OS:      $PRETTY_NAME"
echo "Kernel:  $(uname -r)"
echo "Up:      $(uptime -p), since $(uptime -s)"

section "Load"
read -r l1 l5 l15 _ < /proc/loadavg
echo "Load:    $l1 $l5 $l15 on $(nproc) CPUs"

section "Memory"
awk '/MemTotal/ {t=$2} /MemAvailable/ {a=$2}
     END {printf "Available: %.1f of %.1f GB (%.0f%%)\n", a/1048576, t/1048576, a/t*100}' /proc/meminfo
swapon --noheadings --show=NAME,SIZE,USED | sed 's/^/Swap:    /'

section "Disk over 80%"
df -h -x tmpfs -x devtmpfs -x overlay | awk 'NR > 1 && $5+0 > 80 {print $6, $5; n++} END {if (!n) print "none"}'

section "Failed units"
failed=$(systemctl --failed --no-legend --plain | awk '{print $1}')
echo "${failed:-none}"

section "Listening (non-local TCP)"
ss -tlnH | awk '$4 !~ /^(127\.|\[::1\])/ {print $4}' | sort -u

section "Recent SSH logins"
journalctl -u sshd --no-pager -q 2>/dev/null | grep 'Accepted' | tail -5 | awk '{print $1, $2, $3, $9, "from", $11}'
EOF
chmod +x server-review.sh
sudo ./server-review.sh
```

Install it where Lab 09 will expect it:

```bash
sudo install -o root -g root -m 755 server-review.sh /usr/local/bin/server-review
sudo server-review
```

</details>

## Check yourself

<details><summary>A 4-core server shows a load average of 4.0. Is that a problem?</summary>
Not by itself. It means on average four tasks wanted to run, which four cores can serve with no queue. It is fully used with no headroom. A sustained load well above 4 means work is queueing.
</details>

<details><summary><code>free -h</code> shows 300 MB free on a 64 GB machine. Is it out of memory?</summary>
Look at <code>available</code>. If that is large, the "missing" memory is page cache, which the kernel gives back on demand. Low <code>available</code> together with swap activity would be a real shortage.
</details>

<details><summary><code>df</code> says a filesystem is 100% full, but <code>du</code> adds up to half of that. Two possible explanations?</summary>
A deleted file is still held open by a process (<code>lsof +L1</code>), or data is hidden underneath a mount point: files written to a directory before another filesystem was mounted on top of it.
</details>

<details><summary>What is the difference between a service being active and being enabled?</summary>
Active means it is running now. Enabled means systemd will start it at boot. They are set independently.
</details>

<details><summary>In <code>ss -tlpn</code>, why does it matter whether a service is bound to <code>127.0.0.1</code> or <code>0.0.0.0</code>?</summary>
127.0.0.1 accepts connections only from the machine itself. 0.0.0.0 accepts them on every interface, so only the firewall stands between that service and the network.
</details>

<details><summary>Which file holds SSH login attempts on Rocky? On Ubuntu?</summary>
<code>/var/log/secure</code> on Rocky. <code>/var/log/auth.log</code> on Ubuntu. <code>journalctl -u sshd</code> (or <code>-u ssh</code> on Ubuntu) works on both.
</details>

## Cleanup

```bash
rm -r ~/labwork/06
```

Keep `/usr/local/bin/server-review`; Lab 09 uses it.
