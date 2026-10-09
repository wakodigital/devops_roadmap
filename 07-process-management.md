# Lab 07 — Process Management

**Roadmap topics:** Background / Foreground Processes · Listing / Finding Processes · Process Signals · Killing Processes · Process Priorities · Process Forking
**Time:** about 3 hours
**Touches system state:** installs `strace`, `psmisc` and `tmux`. Every process you start, signal or kill in this lab is one you created. Do not practise `kill` on Docker, Neo4j or the tunnel.

## What you will be able to do

- Run work in the background, bring it back, and keep it alive after you log out.
- Find any process by name, user, port or resource use, and read everything `ps` and `top` report about it.
- Explain what a signal is, send the right one, and write a script that handles them.
- Stop a misbehaving process in the correct order of escalation.
- Explain fork, exec, wait, zombies and orphans, and recognise each in `ps` output.

## Setup

```bash
mkdir -p ~/labwork/07 && cd ~/labwork/07
sudo dnf install -y strace psmisc tmux procps-ng       # psmisc provides pstree and killall
```

Open **two** SSH sessions to the tower. Many exercises run something in one and observe from the other.

---

## 1. What a process is

A **program** is a file on disk. A **process** is a running instance of one: its own memory, open files, environment, working directory, owner and a unique **PID**. Every process has a parent (its **PPID**), which forms a tree rooted at PID 1, `systemd`.

```bash
echo $$                        # PID of your shell
ps -p $$ -o pid,ppid,user,cmd
pstree -p $$                   # your shell and its children
pstree -ps $$                  # the chain of ancestors up to systemd
ls /proc/$$/                   # the kernel's live view of that process
cat /proc/$$/status | head -12
ls -l /proc/$$/cwd /proc/$$/exe
ls -l /proc/$$/fd              # open file descriptors: 0, 1, 2 point at your terminal
tr '\0' '\n' < /proc/$$/environ | head
```

---

## 2. Background and foreground processes

The shell groups the processes of each command line into a **job**. One job at a time is in the **foreground** and owns the terminal. The rest are in the background or stopped.

| Action | How |
|---|---|
| Start in the background | `command &` |
| Suspend the foreground job | `Ctrl-Z` (sends SIGTSTP) |
| List jobs | `jobs` (`jobs -l` adds PIDs) |
| Resume a job in the background | `bg %1` |
| Bring a job to the foreground | `fg %1` |
| Interrupt the foreground job | `Ctrl-C` (sends SIGINT) |
| Signal a job | `kill %1` |
| Wait for background jobs to finish | `wait` |

```bash
sleep 300 &                    # [1] 12345   job number and PID
sleep 400 &
jobs -l
echo $!                        # PID of the most recent background command

sleep 500                      # runs in the foreground; the prompt is gone
# press Ctrl-Z                 # [3]+ Stopped
jobs                           # 1 and 2 Running, 3 Stopped
bg %3                          # resume it in the background
fg %1                          # bring job 1 forward
# press Ctrl-C                 # ends it
kill %2 %3
jobs
```

`%+` or `%%` is the current job, `%-` the previous one, and `%sleep` a job whose command starts with `sleep`.

A background job still writes to your terminal. Redirect its output if you want your prompt left alone:

```bash
(for i in 1 2 3; do echo "noise $i"; sleep 2; done) &            # prints over your prompt
(for i in 1 2 3; do echo "quiet $i"; sleep 2; done) > out.log 2>&1 &
```

### Surviving logout

When your SSH session ends, the shell sends **SIGHUP** ("hangup") to its jobs, and by default that kills them. Four ways around that, from weakest to best:

```bash
# 1. nohup: ignore SIGHUP; output goes to nohup.out unless redirected
nohup sleep 600 > /dev/null 2>&1 &

# 2. disown: remove an already-running job from the shell's job table
sleep 600 &
disown %+                      # or: disown -h %+  to keep it listed but exempt from SIGHUP

# 3. setsid: start it in a new session with no controlling terminal
setsid sleep 600 > /dev/null 2>&1 &

pgrep -a sleep
pkill sleep
```

```bash
# 4. tmux: a terminal that lives on the server
tmux new -s work               # start a named session
# run something long, e.g.:  top
# detach with:  Ctrl-b  then  d
tmux ls
tmux attach -t work            # reattach, from this or any later SSH login
# exit the shell inside, or:  tmux kill-session -t work
```

`tmux` is the right answer for interactive work over SSH on a Wi-Fi-connected server: when the connection drops, everything keeps running and you reattach. For work that should run unattended, the right answer is a systemd service or timer (Lab 09), not a backgrounded shell job.

### Exercises 2

1. Start three `sleep 1000` jobs in the background. Bring the second to the foreground, suspend it, and resume it in the background. Kill all three by job number.
2. Start `sleep 1000 &`, then run `exit` and log back in. Is it still running? (`pgrep -a sleep`) Now repeat with `nohup`. What differs? Clean up.
3. Start a tmux session, run `watch date` in it, detach, close your SSH connection completely, reconnect and reattach.

<details><summary>Solutions</summary>

```bash
# 1
sleep 1000 & sleep 1000 & sleep 1000 &
fg %2            # Ctrl-Z
bg %2
kill %1 %2 %3

# 2
# Whether a plain background job survives depends on the shell's huponexit option
# (shopt huponexit, off by default) and on how the session ended. With a clean "exit" it
# often survives; when the connection drops, it is killed. nohup makes survival certain.
pkill sleep

# 3
tmux new -s t1      # watch date ; Ctrl-b d ; exit ssh ; ssh back in
tmux attach -t t1   # Ctrl-C, then exit
```

</details>

---

## 3. Listing and finding processes

### ps

`ps` takes a snapshot. Two option styles exist for historical reasons; learn one invocation of each.

```bash
ps                              # only processes on this terminal
ps aux                          # BSD style: every process, user-oriented columns
ps -ef                          # System V style: every process, with PPID
ps -ef --forest | less          # as a tree
ps -u $USER                     # one user's processes
ps -p 1 -o pid,ppid,user,stat,etime,cmd       # pick your own columns with -o
ps -eo pid,ppid,user,stat,ni,%cpu,%mem,rss,etime,cmd --sort=-%cpu | head
ps -eo pid,rss,cmd --sort=-rss | head -5
ps -C sshd -o pid,ppid,user,cmd               # by command name
ps -eLf | head                                # threads (LWP column)
```

Columns of `ps aux`:

| Column | Meaning |
|---|---|
| `USER` | owner |
| `PID` | process ID |
| `%CPU` | CPU time divided by lifetime (an average since start, **not** current use) |
| `%MEM` | RSS as a share of physical RAM |
| `VSZ` | virtual size in KB: address space reserved |
| `RSS` | resident set size in KB: physical RAM in use |
| `TTY` | controlling terminal; `?` means none (a daemon) |
| `STAT` | state, see below |
| `START` / `TIME` | when it started / total CPU time consumed |
| `COMMAND` | command line; `[brackets]` mean a kernel thread |

Process states (first letter of `STAT`):

| Code | State |
|---|---|
| `R` | running or runnable |
| `S` | sleeping, waiting for an event (the normal state of most processes) |
| `D` | uninterruptible sleep, usually disk I/O. Counts toward load and cannot be killed until it returns |
| `T` | stopped by a signal or by job control |
| `Z` | zombie: finished, but its parent has not collected its exit status |
| `I` | idle kernel thread |

Extra characters: `s` session leader, `+` foreground process group, `<` high priority, `N` low priority, `l` multi-threaded.

### pgrep and pidof

Do not write `ps aux | grep name`. It matches the `grep` process itself and matches on any part of the line.

```bash
pgrep sshd                      # PIDs whose name matches
pgrep -a sshd                   # with the command line
pgrep -u $USER -l               # everything of yours, with names
pgrep -f "sleep 300"            # -f match against the full command line
pgrep -x bash                   # -x exact name match
pgrep -c sshd                   # count
pgrep -n bash; pgrep -o bash    # newest; oldest
pidof sshd                      # PIDs by exact program name
```

### top

```bash
top
```

Inside top:

| Key | Action |
|---|---|
| `1` | toggle per-CPU lines |
| `P` / `M` / `T` | sort by CPU / memory / CPU time |
| `c` | toggle full command line |
| `V` | tree view |
| `u` | filter by user |
| `k` | send a signal to a PID |
| `r` | renice a PID |
| `H` | show threads |
| `f` | choose columns |
| `W` | save the current layout as default |
| `q` | quit |

```bash
top -b -n 1 | head -15          # batch mode: one snapshot to stdout, usable in scripts
top -p $(pgrep -d, sshd)        # watch specific PIDs
```

`%CPU` in top is per core by default, so a process using four cores fully shows 400%.

### From a port or a file to a process

```bash
sudo ss -tlnp | grep ':22 '     # who is listening on port 22
sudo lsof -i :22                # the same, via lsof (sudo dnf install -y lsof)
sudo lsof -p $$ | head          # everything a process has open
sudo lsof /var/log/messages     # who has this file open
sudo fuser -v /var/log/messages 2>&1 | head
```

### Exercises 3

1. Which process has been running longest on the tower, other than kernel threads? (Hint: `--sort=start_time` or the `etime` column.)
2. Show the five processes using the most physical memory, with RSS in megabytes.
3. Show the full ancestor chain of your current shell.
4. How many processes does each user own? Most first.
5. Start `sleep 777 &`. Find its PID three different ways. Then list its parent PID, state and elapsed time in one `ps` command.
6. Is anything in state `D` or `Z` right now?

<details><summary>Solutions</summary>

```bash
# 1
ps -eo pid,lstart,etime,cmd --sort=start_time | grep -v '\[' | head -3

# 2
ps -eo rss,pid,cmd --sort=-rss | head -6 | awk 'NR > 1 {printf "%7.1f MB  %s  %s\n", $1 / 1024, $2, $3}'

# 3
pstree -ps $$

# 4
ps -eo user --no-headers | sort | uniq -c | sort -rn

# 5
sleep 777 &
echo $!; pgrep -f "sleep 777"; jobs -l; pidof sleep
ps -p $! -o pid,ppid,stat,etime,cmd
kill $!

# 6
ps -eo stat,pid,cmd | awk '$1 ~ /^[DZ]/'
```

</details>

---

## 4. Process signals

A **signal** is a small asynchronous notification delivered to a process by the kernel, on behalf of another process, the terminal, or the kernel itself. The process can **handle** it with its own code, **ignore** it, or let the **default action** happen. Two signals, KILL and STOP, can be neither handled nor ignored.

```bash
kill -l                         # every signal
man 7 signal                    # the reference: default actions and meanings
```

| Signal | No. | Default | Sent by / meaning |
|---|---|---|---|
| `SIGHUP` | 1 | terminate | terminal closed. By convention daemons treat it as "reload your configuration" |
| `SIGINT` | 2 | terminate | `Ctrl-C` |
| `SIGQUIT` | 3 | terminate + core dump | `Ctrl-\` |
| `SIGKILL` | 9 | terminate | **cannot be caught**. The kernel removes the process |
| `SIGSEGV` | 11 | terminate + core dump | the process touched invalid memory |
| `SIGPIPE` | 13 | terminate | wrote to a pipe whose reader has gone (`yes \| head -1`) |
| `SIGTERM` | 15 | terminate | **the polite request to exit**. Default of `kill`, and what `systemctl stop` sends first |
| `SIGCHLD` | 17 | ignore | a child exited or stopped |
| `SIGCONT` | 18 | continue | resume a stopped process (`fg`, `bg`) |
| `SIGSTOP` | 19 | stop | **cannot be caught**. Freezes the process |
| `SIGTSTP` | 20 | stop | `Ctrl-Z`. Can be caught |
| `SIGUSR1`, `SIGUSR2` | 10, 12 | terminate | application-defined |

Signal numbers differ between architectures for some signals. Use names in scripts.

### Watching a process handle signals

```bash
cat > trap.sh <<'EOF'
#!/usr/bin/env bash
trap 'echo "got SIGTERM, cleaning up"; exit 0' TERM
trap 'echo "got SIGHUP, reloading config"' HUP
trap 'echo "got SIGINT, ignoring it"' INT
trap 'echo "got SIGUSR1, status: still running"' USR1
echo "my pid is $$"
while :; do sleep 1; done
EOF
chmod +x trap.sh
```

Terminal 1:

```bash
./trap.sh
```

Terminal 2 (use the PID it printed, or `pgrep -f trap.sh`; `-f` matches any command line containing that text, so close any editor that has the file open):

```bash
P=$(pgrep -f trap.sh)
kill -HUP  $P                  # handled: prints and carries on
kill -USR1 $P
kill -INT  $P                  # handled: Ctrl-C in terminal 1 does the same and no longer stops it
kill -STOP $P                  # frozen. Check: ps -o stat,cmd -p $P  → T
kill -CONT $P                  # resumed
kill -TERM $P                  # handled: cleans up and exits 0
```

Start it again and send `kill -KILL $P`. No message is printed. The script never ran another instruction, so it had no chance to clean up. That is the whole case against `kill -9` as a first resort.

### What signals look like in practice

```bash
sudo systemctl reload sshd      # sends SIGHUP to sshd: re-read config without dropping connections
systemctl show sshd -p ExecReload
sudo kill -USR1 $(pidof dd) 2>/dev/null     # GNU dd prints its progress on SIGUSR1
docker stop <container>         # SIGTERM, then SIGKILL after 10 seconds (illustrative)
```

Exit statuses carry the signal: a process killed by signal N exits with status **128 + N**. So 130 = SIGINT, 137 = SIGKILL (very often the OOM killer, or `docker stop` timing out), 143 = SIGTERM.

```bash
sleep 100 & kill -TERM $!; wait $!; echo "status $?"      # 143
sleep 100 & kill -KILL $!; wait $!; echo "status $?"      # 137
```

Who may signal whom: you can signal your own processes. Root can signal any process.

---

## 5. Killing processes

```bash
kill PID                        # SIGTERM
kill -TERM PID                  # the same, spelled out
kill -15 PID                    # the same, by number
kill -KILL PID                  # SIGKILL
kill -9 PID                     # the same
kill -HUP PID
kill -0 PID                     # sends nothing: tests whether the PID exists and you may signal it
kill -- -PGID                   # a negative number signals a whole process group

pkill sleep                     # by name (same matching rules and options as pgrep)
pkill -u $USER -f "sleep 300"   # by owner and full command line
pkill -TERM -P 1234             # children of PID 1234
killall sleep                   # by exact name
timeout 5 sleep 100             # run a command and SIGTERM it after 5 seconds; exit status 124
timeout -k 2 5 sleep 100        # ...and SIGKILL 2 seconds later if it is still there
```

**Preview a `pkill` with `pgrep -a` using the identical pattern.** `pkill -f python` kills every process with "python" anywhere in its command line.

### The escalation ladder

1. **SIGTERM**, then wait a few seconds. Most programs flush buffers, remove lock files and exit.
2. **SIGINT** or **SIGHUP** if it ignores TERM. Some interactive programs respond to these.
3. **SIGKILL**, last. The process gets no chance to clean up: temp files, lock files, half-written data and shared memory are left behind, and its children are orphaned rather than stopped.

```bash
kill $P; sleep 5; kill -0 $P 2>/dev/null && kill -KILL $P
```

For anything managed by systemd, do not signal it directly. `systemctl stop` runs the same ladder with a configurable timeout, and stops **every** process in the service's cgroup, not just the main PID. `systemctl kill -s HUP unit` sends an arbitrary signal properly.

### Things SIGKILL cannot fix

| Situation | Why | What to do |
|---|---|---|
| A zombie (`Z`) | it is already dead; only an exit record remains | make its parent `wait()`, or kill the parent |
| State `D` | blocked inside the kernel; signals are delivered when it returns | fix the underlying I/O problem |
| It comes straight back | a supervisor restarted it (`Restart=` in systemd, a Docker restart policy) | stop it through the supervisor |

### Exercises 5

1. Start five `sleep 600` processes. Kill exactly two by PID, then the rest with one `pkill`. Preview first.
2. Run `timeout 3 sleep 60; echo $?` and explain the status.
3. Start `./trap.sh`, send it SIGTERM and note the exit status (`echo $?` in terminal 1). Start it again and send SIGKILL. Compare the statuses and explain.
4. Write a one-line loop that waits for a given PID to exit, printing a dot each second. (Hint: `kill -0`.)
5. Run `yes > /dev/null &` twice. Find them in `top`, kill one from inside top with `k`, and the other from the shell.

<details><summary>Solutions</summary>

```bash
# 1
for i in 1 2 3 4 5; do sleep 600 & done
pgrep -a sleep
kill <pid1> <pid2>
pgrep -a -x sleep && pkill -x sleep

# 2
# 124: timeout's own status meaning "the command timed out and I terminated it".

# 3
# SIGTERM → 0, because the trap ran "exit 0".
# SIGKILL → 137 (128 + 9), reported by the shell as "Killed". No handler ran.

# 4
while kill -0 "$P" 2>/dev/null; do printf .; sleep 1; done; echo " gone"

# 5
yes > /dev/null & yes > /dev/null &
top             # k, enter the PID, Enter to accept SIGTERM
pkill -x yes
```

</details>

---

## 6. Process priorities

When more tasks want CPU than there are CPUs, the scheduler shares it out. **Niceness** biases that share.

- Range: **-20** (highest priority) to **19** (lowest). Default 0.
- A "nicer" process yields to others. The effect is only visible when the CPUs are contended; on an idle machine a nice-19 job runs at full speed.
- Any user can **raise** the niceness of their own processes. Only root can lower it or set a negative value.

```bash
nice                            # niceness of the current shell
nice -n 10 sleep 300 &          # start at niceness 10
ps -o pid,ni,pri,cmd -p $!
renice -n 15 -p $!              # change a running process
renice -n 5 -p $!               # FAILS as a normal user: cannot lower it again
sudo renice -n -5 -p $!
sudo renice -n 10 -u $USER      # every process of a user
kill $!
```

In `top`, `NI` is the niceness and `PR` is the kernel's priority (20 + NI for normal tasks; `rt` for real-time).

### See it work

Niceness needs contention, so pin two busy loops to the **same** CPU with `taskset`.

```bash
taskset -c 0 nice -n 0  sh -c 'while :; do :; done' &  A=$!
taskset -c 0 nice -n 19 sh -c 'while :; do :; done' &  B=$!
top -p $A,$B                    # watch %CPU: roughly 98 to 2. Press q.
sudo renice -n 0 -p $B          # sudo, because this lowers the niceness
top -p $A,$B                    # now about 50 / 50
kill $A $B
```

### I/O priority

CPU niceness does nothing for disk contention. `ionice` sets the I/O class, where the I/O scheduler supports it.

```bash
ionice -p $$                    # current class and level
ionice -c 3 tar -czf /tmp/lab-backup.tgz /etc 2>/dev/null     # class 3 = idle: only when nothing else wants the disk
rm -f /tmp/lab-backup.tgz
```

The standard courtesy for a heavy batch job on a shared server is both:

```bash
nice -n 19 ionice -c 3 some-heavy-command        # illustrative
```

For services, set priority in the unit file (`Nice=`, `IOSchedulingClass=`, `CPUWeight=`), and for hard limits use cgroups (Lab 16). Niceness is a hint; a cgroup limit is enforced.

### Exercises 6

1. Start `sleep 500` at niceness 12, verify, raise it to 18, and try to set it back to 12 without sudo.
2. Which processes on the tower run at a non-zero niceness?
3. Repeat the two-loop experiment with niceness 0 and 5. Roughly what split do you see?

<details><summary>Solutions</summary>

```bash
# 1
nice -n 12 sleep 500 &
ps -o ni,cmd -p $!
renice -n 18 -p $!
renice -n 12 -p $!          # Permission denied
kill $!

# 2
ps -eo ni,pid,cmd | awk '$1 != 0 && $1 != "-" && NR > 1'

# 3
# Around 75 / 25. Each niceness step changes the weight by about 1.25x,
# so five steps is roughly a 3:1 ratio.
```

</details>

---

## 7. Process forking

Unix creates processes in two steps.

1. **`fork()`** duplicates the calling process. The child is a copy: same code, same memory contents (shared copy-on-write), same open files, same environment. The only difference is the return value: 0 in the child, the child's PID in the parent.
2. **`exec()`** replaces the current process's program with a new one. Same PID, new code.

When you type `ls`, bash forks a copy of itself, the copy execs `/usr/bin/ls`, and the original bash calls **`wait()`** to sleep until the child exits and to collect its exit status. That status is what you read in `$?`.

This design is why children inherit environment variables and open files (Lab 03), and why redirection works: the child sets up its file descriptors after the fork and before the exec, so `ls` never knows its stdout is a file.

### Watch it happen

```bash
strace -f -e trace=clone,clone3,fork,vfork,execve,wait4,exit_group bash -c 'ls > /dev/null; echo done' 2>&1 | grep -v '+++'
```

You will see `execve("/usr/bin/bash"…)`, then a `clone` returning a new PID, then in the child `execve("/usr/bin/ls"…)`, then `exit_group(0)`, then the parent's `wait4` returning that PID. (Modern glibc implements `fork()` with the `clone` system call.)

### Subshells and exec in bash

```bash
echo "shell: $$"
( echo "subshell: $BASHPID, parent $PPID, \$\$ still says $$" )    # ( ) forks a subshell
x=1; ( x=2 ); echo "x=$x"                                           # 1: the child changed its own copy
echo hi | read v; echo "v=[$v]"                                     # empty: each pipeline stage is a forked child

bash -c 'echo "before exec: $$"; exec sleep 3' &                    # exec: same PID becomes sleep
sleep 0.5; ps -o pid,cmd -p $!; wait
```

`exec` at the end of a wrapper script is standard in container entrypoints: the application replaces the shell and becomes PID 1, so it receives signals from `docker stop` directly.

### Zombies

When a process exits, the kernel frees its memory but keeps a small record (PID and exit status) until the parent calls `wait()`. In that window the process is a **zombie**, shown as `Z` and `<defunct>`. A zombie uses no CPU or memory. It holds one entry in the process table.

Create one. The outer process starts a short-lived child and then execs into `sleep`, a program that never calls `wait()`:

```bash
( sleep 1 & exec sleep 30 ) &
sleep 2
ps -eo pid,ppid,stat,cmd | grep -E '[s]leep|defunct'
```

```
  5210  4801 S    sleep 30
  5211  5210 Z    [sleep] <defunct>
```

```bash
kill -9 5211                    # use your zombie's PID: nothing happens. It is already dead.
ps -o pid,stat,cmd -p 5211
```

The cure is the parent. When the `sleep 30` finishes (or you kill it), the zombie is re-parented to PID 1, and systemd reaps it immediately.

A handful of zombies is harmless. A steadily growing number means the parent has a bug, and eventually the PID table fills.

```bash
ps -eo stat | grep -c '^Z'                                 # count zombies
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {print $2}' | sort | uniq -c     # which parents own them
cat /proc/sys/kernel/pid_max
```

### Orphans

If a parent exits first, its children are **orphans**. They are adopted by PID 1 (or by the nearest "subreaper", such as your user's `systemd --user` instance or a container's init), which will reap them when they exit.

```bash
bash -c 'sleep 60 & echo "child is $!"'       # the bash exits at once; the sleep lives on
ps -o pid,ppid,cmd -p $(pgrep -n sleep)       # PPID is now 1, or your systemd --user PID
pkill -n sleep
```

This is the mechanism behind daemons, and behind `nohup`/`setsid` in section 2.

In containers this matters: PID 1 inside a container is your application, and most applications do not reap adopted children. That is what `docker run --init` (which adds a tiny init called tini) is for.

### Limits on forking

```bash
ulimit -u                                   # maximum processes for your user
cat /proc/sys/kernel/threads-max
systemctl show user-$(id -u).slice -p TasksMax
```

A **fork bomb** is a process that forks without limit until nothing else can start. These limits, and the cgroup `pids` controller in Lab 16, are the defence. Do not test one on the tower.

### Exercises 7

1. Use `strace` as above on `bash -c 'date; date; echo end'`. How many `clone` calls and how many `execve` calls? Why is there one more `execve` than `clone`? Then drop the `; echo end` and run it again. What changed?
2. Predict the output: `x=outer; ( x=inner; echo $x ); echo $x`. Explain it in terms of fork.
3. Why does `echo hello | read word; echo $word` print an empty line in bash? Give a version that works.
4. Create a zombie, identify its parent, and make it disappear without waiting 30 seconds.
5. From `pstree -p`, find your SSH session. Which process forked your shell, and which forked that?

<details><summary>Solutions</summary>

```bash
# 1
# 2 clones, 3 execves. The first execve is strace starting bash itself; then each date is one
# fork (clone) plus one execve. echo is a builtin, so it needs neither.
# Without "; echo end": 1 clone, 3 execves. bash sees that the second date is the last thing it
# will ever do, so it skips the fork and execs date in its own process.

# 2
# inner, then outer. The parentheses fork a child with a copy of x. The child changes its copy and exits.

# 3
# read runs in a forked child of the pipeline; its variable dies with it.
read word <<< "hello"; echo $word
# or: word=$(echo hello)

# 4
( sleep 1 & exec sleep 30 ) &
sleep 2
ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/'      # column 2 is the parent
kill <ppid>                                      # the parent dies, PID 1 adopts and reaps the zombie

# 5
pstree -ps $$
# systemd ── sshd (listener) ── sshd (your session, root) ── sshd (your session, unprivileged) ── bash
```

</details>

---

## Challenge: the runaway job

Simulate an incident and handle it the way you would on a production box. Terminal 1 plays the faulty application.

```bash
cat > runaway.sh <<'EOF'
#!/usr/bin/env bash
trap '' TERM INT HUP                 # ignores the polite signals
for i in 1 2 3; do ( while :; do :; done ) & done
wait
EOF
chmod +x runaway.sh
./runaway.sh
```

From terminal 2, without looking at the script again:

1. Notice the problem from `uptime` and `top`.
2. Identify the parent and all its children, as a tree.
3. Reduce the impact first: renice all of them to 19.
4. Freeze them with SIGSTOP and confirm CPU use has dropped. Why might you do this before killing?
5. Try SIGTERM on the parent. Confirm it was ignored.
6. Kill the whole family in a way that leaves no orphaned busy loops behind.
7. Confirm nothing is left.

<details><summary>Solution</summary>

```bash
uptime; top -b -n 1 | head -12                       # 1
P=$(pgrep -f runaway.sh | head -1); pstree -p $P     # 2
renice -n 19 -p $P $(pgrep -P $P)                    # 3
kill -STOP $P $(pgrep -P $P); top -b -n 1 | head -12 # 4: freezing stops the damage and preserves the
                                                     #    processes for inspection (/proc/PID, strace, gdb)
kill -CONT $P; kill -TERM $P; sleep 1; kill -0 $P && echo "still alive"   # 5
# 6: the children are in the parent's process group. Signal the group.
PGID=$(ps -o pgid= -p $P | tr -d ' ')
kill -KILL -- -$PGID
pgrep -af runaway || echo "clean"                    # 7
```

Killing only the parent would have orphaned three busy loops, re-parented to PID 1 and still burning CPU. Signalling the process group (or, for a service, `systemctl stop`, which kills the whole cgroup) is how you get everything.

</details>

## Check yourself

<details><summary>What is the difference between SIGTERM and SIGKILL?</summary>
SIGTERM is delivered to the process, which can catch it, clean up and exit, or ignore it. SIGKILL is never delivered to the process's code at all; the kernel ends it immediately and nothing gets cleaned up.
</details>

<details><summary>Why can you not kill a zombie?</summary>
It has already exited. What remains is an entry in the process table holding its exit status, waiting for the parent to read it. Only the parent calling <code>wait()</code>, or the parent exiting so that PID 1 inherits and reaps it, removes the entry.
</details>

<details><summary>A container exits with status 137. What happened?</summary>
137 = 128 + 9: it was ended by SIGKILL. Usually the OOM killer because it exceeded its memory limit, or <code>docker stop</code> escalating after the 10-second grace period.
</details>

<details><summary>What does <code>nice -n 19</code> do on a machine with idle CPUs?</summary>
Effectively nothing. Niceness only decides who wins when processes compete for the same CPU.
</details>

<details><summary>You run <code>kill 1234</code> and the process is still there a minute later. Name three possible reasons.</summary>
It traps or ignores SIGTERM; it is in uninterruptible sleep (state D) and has not returned to user space; or it did exit and a supervisor restarted it under a new PID. A fourth: it is a zombie.
</details>

<details><summary>Why is <code>ps aux | grep nginx</code> a poor way to find a process?</summary>
It matches the grep command itself and any process with "nginx" anywhere in its command line or username. <code>pgrep -a nginx</code> or <code>pgrep -x nginx</code> matches the process name.
</details>

## Cleanup

```bash
pkill -u $USER -x sleep; pkill -u $USER -x yes; pkill -f runaway.sh; pkill -f trap.sh
tmux kill-server 2>/dev/null
rm -r ~/labwork/07
```
