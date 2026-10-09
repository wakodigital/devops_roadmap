# Lab 09 — Service Management (systemd)

**Roadmap topics:** Checking Service Status · Starting / Stopping Services · Checking Service Logs · Creating New Services
**Time:** about 3.5 hours
**Touches system state:** yes. Creates units named `lab-*` in `/etc/systemd/system`, scripts named `lab-*` in `/usr/local/bin`, and the `labsvc` system account. Existing services are only read, never stopped or reconfigured. Cleanup removes everything.

## What you will be able to do

- Read `systemctl status` output completely.
- Control services and explain the difference between start, enable, reload, restart and mask.
- Pull exactly the log lines you need out of the journal.
- Write a unit file for your own program, run it as an unprivileged user, make it restart on failure, and harden it.
- Schedule a job with a systemd timer.
- Diagnose a unit that will not start.

## Background

`systemd` is PID 1. It starts everything else at boot, supervises it afterwards, and collects its output. It manages **units**, each described by a small INI-style file. The unit type is the file extension.

| Type | Purpose |
|---|---|
| `.service` | a process or group of processes to run and supervise |
| `.timer` | schedule the activation of another unit (systemd's cron) |
| `.socket` | listen on a socket and start a service when a connection arrives |
| `.target` | a named group of units, used as a synchronisation point (`multi-user.target`) |
| `.mount`, `.automount` | filesystems |
| `.path` | start a unit when a file or directory changes |
| `.slice`, `.scope` | cgroup groupings for resource control (Lab 16) |

Unit files are searched in this order, first match wins:

| Path | Whose | Edit? |
|---|---|---|
| `/etc/systemd/system/` | the administrator's | **yes, your units and overrides go here** |
| `/run/systemd/system/` | runtime, lost at reboot | rarely |
| `/usr/lib/systemd/system/` | installed by packages | **no**, updates overwrite it |

## Setup

```bash
mkdir -p ~/labwork/09 && cd ~/labwork/09

# A trivial service to practise on, so that no real service is disturbed
sudo tee /etc/systemd/system/lab-sleeper.service > /dev/null <<'EOF'
[Unit]
Description=Lab sleeper (does nothing, safely)

[Service]
ExecStart=/usr/bin/sleep infinity

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
```

---

## 1. Checking service status

```bash
systemctl status sshd
```

```
● sshd.service - OpenSSH server daemon
     Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-10-05 08:12:44 MDT; 3 days ago
       Docs: man:sshd(8)
   Main PID: 912 (sshd)
      Tasks: 1 (limit: 409326)
     Memory: 5.4M
        CPU: 1.204s
     CGroup: /system.slice/sshd.service
             └─912 "sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups"

Oct 08 20:15:01 tower sshd[48211]: Accepted publickey for walter from 10.0.0.5 port 51844 ssh2
```

| Line | What it tells you |
|---|---|
| `●` | colour: green active, white inactive, red failed |
| `Loaded:` | which unit file was read; whether it is **enabled** (starts at boot); the vendor's preset |
| `Active:` | the current state and sub-state, and since when. The timestamp shows whether it restarted recently |
| `Main PID:` | the process systemd considers the service |
| `Tasks:` / `Memory:` / `CPU:` | resources used by the whole service, from its cgroup |
| `CGroup:` | **every** process belonging to the service, including children |
| log lines | the last ten journal lines for the unit. `-n 50` shows more, `-l` stops truncation |

Active states:

| State | Meaning |
|---|---|
| `active (running)` | a process is running |
| `active (exited)` | a one-shot job ran successfully and there is nothing left running. Normal for setup tasks |
| `active (waiting)` | waiting for an event (timers, sockets) |
| `inactive (dead)` | not running, and did not fail |
| `failed` | exited with an error, crashed, timed out, or hit its restart limit |
| `activating` / `deactivating` | in transition. Stuck here = start or stop is hanging |

### Scripting-friendly queries

```bash
systemctl is-active sshd          # prints the state; exit status 0 only if active
systemctl is-enabled sshd
systemctl is-failed sshd
systemctl is-active --quiet docker && echo "docker is up"
systemctl show sshd -p MainPID -p ActiveState -p SubState -p ExecMainStartTimestamp -p NRestarts
systemctl show sshd | wc -l       # every property systemd holds for the unit
```

`is-enabled` answers:

| Output | Meaning |
|---|---|
| `enabled` | starts at boot |
| `disabled` | does not, but can be started by hand |
| `static` | has no `[Install]` section; cannot be enabled, only pulled in by another unit |
| `masked` | linked to `/dev/null`; cannot be started at all |
| `indirect`, `alias`, `generated` | enabled through another unit, an alias, or created by a generator |

### Looking across the system

```bash
systemctl list-units --type=service                    # loaded and active
systemctl list-units --type=service --all              # including inactive
systemctl list-units --type=service --state=failed     # same as: systemctl --failed
systemctl list-unit-files --type=service               # every installed unit file and its enablement
systemctl list-dependencies sshd                       # what sshd pulls in
systemctl list-dependencies --reverse sshd             # what pulls sshd in
systemctl list-dependencies multi-user.target | head -30
systemctl get-default                                  # the target the machine boots into
systemctl cat sshd                                     # the unit file, plus any drop-in overrides
systemctl status 1                                     # status by PID: which unit owns that process?
systemctl status                                       # the whole machine, as a tree
```

### Exercises 1

1. Is `docker` active? Is it enabled? Since when has it been running, and what is its main PID?
2. Which unit file defines `crond`, and what command does it run?
3. How many services are enabled on the tower? How many are `static`?
4. Which unit owns PID 1's first child in `pstree`? Use `systemctl status <pid>`.
5. What does `systemctl status lab-sleeper` say right now? Explain each of `Loaded` and `Active`.

<details><summary>Solutions</summary>

```bash
# 1
systemctl is-active docker; systemctl is-enabled docker
systemctl show docker -p ActiveEnterTimestamp -p MainPID

# 2
systemctl cat crond | head -3          # first line is a comment giving the path
systemctl show crond -p ExecStart

# 3
systemctl list-unit-files --type=service --state=enabled --no-legend | wc -l
systemctl list-unit-files --type=service --state=static  --no-legend | wc -l

# 4
pstree -p 1 | head -3; systemctl status <pid> | head -3

# 5
# Loaded: loaded (/etc/systemd/system/lab-sleeper.service; disabled; ...)  → file read, not enabled
# Active: inactive (dead)                                                  → never started
```

</details>

---

## 2. Starting and stopping services

Practise on `lab-sleeper`.

```bash
sudo systemctl start lab-sleeper
systemctl status lab-sleeper
sudo systemctl stop lab-sleeper
sudo systemctl restart lab-sleeper          # stop then start; starts it if it was stopped
sudo systemctl try-restart lab-sleeper      # restart only if it is already running
systemctl show lab-sleeper -p MainPID       # a new PID after each restart
```

| Command | Effect | Survives reboot? |
|---|---|---|
| `start` / `stop` | run / end it now | no |
| `restart` | stop, then start. Connections drop | no |
| `reload` | ask the running process to re-read its configuration (the unit's `ExecReload`, usually SIGHUP). No downtime. Only works if the service supports it | no |
| `reload-or-restart` | reload if supported, else restart | no |
| `enable` / `disable` | create / remove the symlink that makes it start at boot. **Does not start or stop it now** | yes |
| `enable --now` / `disable --now` | both at once | yes |
| `mask` / `unmask` | make it impossible to start, even as a dependency / undo | yes |
| `kill -s SIG` | send a signal to all of the unit's processes | no |
| `reset-failed` | clear a `failed` state and the restart counter | |

### start and enable are independent

```bash
systemctl is-active lab-sleeper; systemctl is-enabled lab-sleeper      # active, disabled

sudo systemctl enable lab-sleeper
```

Read what it printed:

```
Created symlink /etc/systemd/system/multi-user.target.wants/lab-sleeper.service → /etc/systemd/system/lab-sleeper.service.
```

That symlink is the entire meaning of "enabled". The unit's `[Install]` section says `WantedBy=multi-user.target`, so enabling links it into that target's `.wants` directory, and when the machine reaches `multi-user.target` at boot, the service is pulled in.

```bash
ls -l /etc/systemd/system/multi-user.target.wants/ | grep lab
sudo systemctl disable lab-sleeper          # removes the symlink; the service keeps running
systemctl is-active lab-sleeper; systemctl is-enabled lab-sleeper
sudo systemctl enable --now lab-sleeper     # the usual form when installing something
```

Check the four combinations whenever you set up a service. "Running but not enabled" works until the next reboot. "Enabled but not running" means it failed, or was never started.

### mask

`disable` stops a unit starting at boot, but anything can still start it by hand or as a dependency. `mask` makes it impossible to start by any route, by linking the unit's name to `/dev/null` in `/etc/systemd/system`, which shadows the real unit file in `/usr/lib`.

Practise on `ctrl-alt-del.target`, the unit that reboots the machine when someone presses Ctrl-Alt-Del on an attached keyboard. Masking it is a common hardening step and harmless on a headless server.

```bash
systemctl is-enabled ctrl-alt-del.target        # if this already says "masked", your hardening did it: read along and skip the commands
sudo systemctl mask ctrl-alt-del.target
ls -l /etc/systemd/system/ctrl-alt-del.target   # -> /dev/null
systemctl is-enabled ctrl-alt-del.target        # masked
sudo systemctl start ctrl-alt-del.target        # refused: "Unit ctrl-alt-del.target is masked."
sudo systemctl unmask ctrl-alt-del.target       # removes the link
```

`mask` cannot be used on a unit whose file is itself in `/etc/systemd/system` (such as `lab-sleeper`): the link would have to replace your file, and systemd refuses. Use mask when a packaged service must not come back, such as a second firewall or time daemon that conflicts with the one you chose.

### daemon-reload

systemd reads unit files into memory. After you create or edit one **on disk**, tell systemd to re-read them:

```bash
sudo systemctl daemon-reload
```

It does not restart anything. It is unrelated to `reload`, which is about a service's own configuration. Forgetting it produces the warning `unit file changed on disk`.

### Stopping, in detail

`systemctl stop` sends SIGTERM to the processes in the unit's cgroup, waits up to `TimeoutStopSec` (90 seconds by default), then sends SIGKILL to whatever remains. This is the escalation ladder from Lab 07, applied to the whole process tree.

```bash
systemctl show lab-sleeper -p KillMode -p KillSignal -p TimeoutStopUSec
```

### Targets and power

```bash
systemctl get-default                       # multi-user.target on a headless server
systemctl list-units --type=target
# sudo systemctl isolate rescue.target      # switch target NOW: drops you to single-user mode. Not over SSH.
# sudo systemctl reboot / poweroff          # the supported ways to restart or shut down
```

### Exercises 2

1. Put `lab-sleeper` into each of the four states (running/stopped × enabled/disabled) in turn, checking with `is-active` and `is-enabled` each time. Finish with it running and enabled.
2. Run `sudo systemctl reload lab-sleeper`. What happens, and why?
3. Send the service SIGTERM using `systemctl kill`. What state is it in afterwards? Does systemd restart it?
4. Without restarting anything, list which enabled services are **not** currently active. Are any of them a problem?

<details><summary>Solutions</summary>

```bash
# 1
sudo systemctl enable --now lab-sleeper
sudo systemctl stop lab-sleeper
sudo systemctl disable lab-sleeper
sudo systemctl start lab-sleeper
sudo systemctl enable lab-sleeper

# 2
# "Failed to reload lab-sleeper.service: Job type reload is not applicable for unit ..."
# The unit has no ExecReload=, so systemd has no way to ask it to reload.

# 3
sudo systemctl kill -s TERM lab-sleeper; systemctl status lab-sleeper | sed -n 3p
# inactive (dead): systemd counts death by SIGTERM as a clean exit. It stays down, because the unit has no Restart= setting.
sudo systemctl start lab-sleeper

# 4
for u in $(systemctl list-unit-files --type=service --state=enabled --no-legend | awk '{print $1}'); do
  systemctl is-active --quiet "$u" || echo "$u: $(systemctl is-active "$u")"
done
# One-shot units that ran at boot and exited show "inactive" legitimately. "failed" is the problem state.
```

</details>

---

## 3. Checking service logs

systemd captures everything a service writes to stdout and stderr, plus syslog messages and kernel messages, into the **journal**: a structured, indexed, binary log read with `journalctl`.

```bash
sudo journalctl -u sshd                     # one unit, oldest first, in a pager
sudo journalctl -u sshd -e                  # jump to the end
sudo journalctl -u sshd -n 30 --no-pager    # last 30 lines, no pager (for scripts and pasting)
sudo journalctl -u sshd -f                  # follow, like tail -f
sudo journalctl -u sshd -r                  # newest first
sudo journalctl -u docker -u sshd           # several units, interleaved by time
```

You need `sudo` (or membership of `systemd-journal` or `wheel`) to see other users' and system messages.

### Filtering by time

```bash
sudo journalctl -u sshd --since today
sudo journalctl -u sshd --since "1 hour ago"
sudo journalctl --since "2026-10-08 09:00" --until "2026-10-08 09:30"
sudo journalctl --since yesterday --until today
sudo journalctl -b                          # this boot
sudo journalctl -b -1                       # the previous boot
sudo journalctl --list-boots
```

### Filtering by priority

Syslog levels: 0 `emerg`, 1 `alert`, 2 `crit`, 3 `err`, 4 `warning`, 5 `notice`, 6 `info`, 7 `debug`. `-p` shows that level **and everything more severe**.

```bash
sudo journalctl -p err -b                   # errors and worse since boot: the first thing to read on a sick machine
sudo journalctl -p warning --since today
sudo journalctl -p err..crit -b             # a range
sudo journalctl -u docker -p err --since "24 hours ago"
```

### Filtering by anything else

```bash
sudo journalctl -k                          # kernel messages (like dmesg), -k -b -1 for the previous boot
sudo journalctl -g 'fail|error' -u sshd     # grep with a regex (case-insensitive if the pattern is all lowercase)
sudo journalctl _PID=1 -n 5
sudo journalctl _UID=$(id -u) --since today
sudo journalctl _COMM=sudo --since today
sudo journalctl /usr/sbin/sshd -n 5         # by executable path
sudo journalctl -t lab-test                 # by syslog identifier (tag)
```

### Output formats

```bash
sudo journalctl -u sshd -n 3 -o short-iso           # ISO timestamps
sudo journalctl -u sshd -n 3 -o short-precise       # microseconds
sudo journalctl -u sshd -n 1 -o json-pretty         # every stored field
sudo journalctl -u sshd -n 3 -o cat                 # the message only
sudo journalctl -u sshd -n 3 -x                     # add explanatory help text where available
sudo journalctl -u sshd -o json --since today | jq -r '.MESSAGE' | head     # if jq is installed
```

`-o json-pretty` shows what makes the journal different from a text file: every entry carries trusted metadata (`_PID`, `_UID`, `_SYSTEMD_UNIT`, `_BOOT_ID`) that the logging process cannot forge.

### Writing to the journal yourself

```bash
logger -t lab-test "hello from the shell"
logger -t lab-test -p user.err "this one is an error"
sudo journalctl -t lab-test
sudo journalctl -t lab-test -p err
echo "from systemd-cat" | systemd-cat -t lab-test -p warning
```

Scripts run by cron or timers should log this way, or simply print to stdout and let systemd capture it.

### Storage

```bash
sudo journalctl --disk-usage
ls -d /var/log/journal 2>/dev/null && echo "persistent: logs survive reboot" || echo "volatile: kept in /run, lost at reboot"
grep -vE '^\s*(#|$)' /etc/systemd/journald.conf
sudo journalctl --vacuum-time=30d           # delete entries older than 30 days (this really deletes)
sudo journalctl --vacuum-size=500M          # or shrink to a size
```

Limits are set in `/etc/systemd/journald.conf` (`SystemMaxUse=`, `MaxRetentionSec=`). On Rocky, `rsyslog` also runs and writes the traditional text files (`/var/log/messages`, `/var/log/secure`), which `logrotate` rotates. Both hold the same events.

### Exercises 3

1. Show every log line from `lab-sleeper` since you created it. How many times was it started?
2. Show all messages of priority `err` or worse from the current boot. Pick one and work out which unit produced it.
3. What was logged system-wide in the 60 seconds after the last boot started? (Hint: `--list-boots`, `-b`, `--until`.)
4. Follow the journal for `lab-sleeper` in one terminal. In another, restart it. What lines appear, and who wrote them: the service or systemd?
5. Print today's sudo commands as bare message text, no timestamps.

<details><summary>Solutions</summary>

```bash
# 1
sudo journalctl -u lab-sleeper --no-pager
sudo journalctl -u lab-sleeper --no-pager | grep -c 'Started'

# 2
sudo journalctl -p err -b --no-pager | tail -20
sudo journalctl -p err -b -o json -n 1 | jq -r '._SYSTEMD_UNIT, .MESSAGE'      # or read -o verbose

# 3
sudo journalctl -b --no-pager | head -60
# precise version: take the boot start time from --list-boots and add a minute
sudo journalctl -b --until "$(date -d "$(uptime -s) + 1 minute" '+%F %T')" --no-pager | tail

# 4
sudo journalctl -u lab-sleeper -f
# "Stopping...", "Deactivated successfully.", "Stopped...", "Started..." all come from systemd (PID 1).
# sleep itself prints nothing.

# 5
sudo journalctl _COMM=sudo --since today -o cat | grep COMMAND
```

</details>

---

## 4. Creating new services

You will write a small program, then a unit that runs it properly: as an unprivileged user, restarted on failure, logging to the journal, with its own state directory.

### The program

```bash
sudo tee /usr/local/bin/lab-heartbeat > /dev/null <<'EOF'
#!/usr/bin/env bash
# Prints a heartbeat line every HEARTBEAT_INTERVAL seconds and records it in a state file.

interval="${HEARTBEAT_INTERVAL:-5}"
state="${STATE_DIRECTORY:-/tmp}/beats.log"

trap 'echo "SIGTERM received, shutting down cleanly"; exit 0' TERM
trap 'echo "SIGHUP received, would reload configuration here"' HUP

echo "starting: pid $$, user $(id -un), interval ${interval}s, state file $state"
n=0
while :; do
  n=$((n + 1))
  echo "heartbeat $n"
  echo "$(date -Is) $n" >> "$state"
  sleep "$interval" &
  wait $!            # wait is interrupted by signals, so traps run at once
done
EOF
sudo chmod 755 /usr/local/bin/lab-heartbeat

/usr/local/bin/lab-heartbeat          # test it by hand first. Ctrl-C to stop.
```

Always run the program by hand before writing a unit for it. It separates "my program is broken" from "my unit is wrong".

**Why `/usr/local/bin` and not your home directory:** on Rocky, SELinux labels files by location. Files under `/home` are labelled `user_home_t`, which systemd is not permitted to execute, and the service fails with `status=203/EXEC` even though the mode bits are fine. Exercise 5 reproduces this.

### The account

```bash
id labsvc 2>/dev/null || sudo useradd -r -s /sbin/nologin -c "Lab service account" labsvc
```

### The unit

```bash
sudo tee /etc/systemd/system/lab-heartbeat.service > /dev/null <<'EOF'
[Unit]
Description=Lab heartbeat service
Documentation=file:///usr/local/bin/lab-heartbeat
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=labsvc
Group=labsvc
Environment=HEARTBEAT_INTERVAL=5
StateDirectory=lab-heartbeat
ExecStart=/usr/local/bin/lab-heartbeat
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemd-analyze verify /etc/systemd/system/lab-heartbeat.service     # nothing about lab-heartbeat = no problems found
sudo systemctl daemon-reload
sudo systemctl enable --now lab-heartbeat
systemctl status lab-heartbeat
sudo journalctl -u lab-heartbeat -f          # Ctrl-C after a few beats
sudo ls -l /var/lib/lab-heartbeat/           # created by StateDirectory=, owned by labsvc
```

### What each directive does

`[Unit]`: identity and ordering.

| Directive | Meaning |
|---|---|
| `Description=` | shown in status and logs |
| `After=` / `Before=` | **ordering only**: if both are being started, start this one after / before the other |
| `Wants=` | **weak dependency**: also start the other unit; carry on if it fails |
| `Requires=` | **strong dependency**: if the other fails to start, this one is not started |
| `BindsTo=`, `PartOf=` | stop or restart together with the other unit |

`After=` and `Wants=` are separate questions (order versus pulling in) and you usually need both.

`[Service]`: how to run it.

| Directive | Meaning |
|---|---|
| `Type=simple` | (default) the process started by `ExecStart` is the service and stays in the foreground |
| `Type=exec` | like simple, but "started" only once the binary has actually been executed |
| `Type=forking` | the program daemonises itself; systemd waits for the parent to exit. Needs `PIDFile=`. For legacy daemons |
| `Type=oneshot` | runs to completion and exits; systemd waits for it. For scripts and timer jobs. Often with `RemainAfterExit=yes` |
| `Type=notify` | the program tells systemd when it is ready (`sd_notify`) |
| `ExecStart=` | the command. **Absolute path required.** Not run through a shell: no pipes, redirects or `&&` unless you write `/bin/bash -c '…'` |
| `ExecStartPre=` / `ExecStartPost=` | commands before / after. A `-` prefix on the path means "failure is acceptable" |
| `ExecReload=` | what `systemctl reload` runs |
| `ExecStop=` | a custom stop command. Default is SIGTERM |
| `User=` / `Group=` | run as this account, not root |
| `WorkingDirectory=` | cwd for the process |
| `Environment=` / `EnvironmentFile=` | variables, inline or from a `KEY=value` file (keep secrets in a root-owned 600 file) |
| `Restart=` | `no`, `on-failure`, `always`, `on-abnormal`… |
| `RestartSec=` | delay between restarts |
| `StateDirectory=` | create `/var/lib/NAME`, owned by `User=`, and export `$STATE_DIRECTORY` |
| `LogsDirectory=`, `CacheDirectory=`, `RuntimeDirectory=`, `ConfigurationDirectory=` | the same for `/var/log`, `/var/cache`, `/run`, `/etc` |
| `StandardOutput=` / `StandardError=` | `journal` (default), `append:/path`, `null` |
| `TimeoutStartSec=` / `TimeoutStopSec=` | how long to wait before treating a start or stop as failed |

`[Install]`: what `enable` should do.

| Directive | Meaning |
|---|---|
| `WantedBy=multi-user.target` | start at normal boot. Use this for almost every service on a server |

### Prove the supervision works

```bash
systemctl show lab-heartbeat -p MainPID -p NRestarts
sudo kill -9 $(systemctl show lab-heartbeat -p MainPID --value)     # simulate a crash
sleep 4
systemctl status lab-heartbeat | head -6                            # active again, with a new PID
systemctl show lab-heartbeat -p NRestarts
sudo journalctl -u lab-heartbeat -n 8 --no-pager                    # "Main process exited, code=killed, status=9/KILL"
                                                                    # "Scheduled restart job, restart counter is at 1."

sudo systemctl reload lab-heartbeat                                 # ExecReload → SIGHUP → the trap
sudo journalctl -u lab-heartbeat -n 3 --no-pager

sudo systemctl stop lab-heartbeat                                   # SIGTERM → trap → exit 0
sudo journalctl -u lab-heartbeat -n 4 --no-pager                    # "shutting down cleanly", no restart
sudo systemctl start lab-heartbeat
```

`Restart=on-failure` restarted it after the kill and correctly left it alone after a clean stop.

`Restart=` values:

| Value | Restarts after |
|---|---|
| `no` | never |
| `on-failure` | non-zero exit, a signal, a timeout. **The usual choice** |
| `always` | any exit, including a clean one (but not `systemctl stop`) |
| `on-abnormal` | signal or timeout, but not a non-zero exit |

A unit that keeps failing is stopped from thrashing: after `StartLimitBurst=` (5) starts within `StartLimitIntervalSec=` (10 s), systemd gives up with `start-limit-hit`. `sudo systemctl reset-failed UNIT` clears that.

### Changing a unit: drop-ins

To change one setting, do not edit the whole unit, and never edit a packaged unit under `/usr/lib`. Add a **drop-in** that overrides just what you need.

```bash
sudo systemctl edit lab-heartbeat
```

An editor opens. Between the marker comments, enter:

```ini
[Service]
Environment=HEARTBEAT_INTERVAL=2
```

Save and quit. `systemctl edit` writes `/etc/systemd/system/lab-heartbeat.service.d/override.conf` and runs `daemon-reload` for you.

```bash
systemctl cat lab-heartbeat                 # the unit followed by the drop-in
sudo systemctl restart lab-heartbeat        # a changed unit only takes effect on restart
sudo journalctl -u lab-heartbeat -n 2 --no-pager      # "interval 2s"
systemd-delta --type=extended               # every unit on the system that has been overridden
sudo systemctl revert lab-heartbeat         # remove all drop-ins and return to the base unit
sudo systemctl restart lab-heartbeat
```

Settings that are lists (`ExecStart=`, `Environment=` are additive in different ways) sometimes need clearing first: to replace `ExecStart` in a drop-in, write an empty `ExecStart=` line and then the new one.

This is how you would, for example, raise `LimitNOFILE` for Docker or add `Restart=` to a packaged service without your change being lost at the next `dnf upgrade`.

### Hardening

systemd can sandbox a service with one line per restriction.

```bash
systemd-analyze security lab-heartbeat | tail -3     # an exposure score from 0 (locked down) to 10
systemd-analyze security lab-heartbeat | head -40    # each ✗ is a restriction not yet applied
```

```bash
sudo systemctl edit lab-heartbeat
```

```ini
[Service]
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
RestrictNamespaces=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
CapabilityBoundingSet=
MemoryMax=64M
CPUQuota=20%
```

```bash
sudo systemctl restart lab-heartbeat
systemctl status lab-heartbeat | head -8             # still works
systemd-analyze security lab-heartbeat | tail -3     # a much lower score
```

| Directive | Effect |
|---|---|
| `NoNewPrivileges=yes` | the process and its children can never gain privileges (setuid binaries stop working) |
| `ProtectSystem=strict` | the whole filesystem is read-only to the service, except `/dev`, `/proc`, `/sys` and directories granted through `StateDirectory=` or `ReadWritePaths=` |
| `ProtectHome=yes` | `/home`, `/root` and `/run/user` appear empty |
| `PrivateTmp=yes` | a private `/tmp` and `/var/tmp`, invisible to other services |
| `PrivateDevices=yes` | no access to physical devices |
| `CapabilityBoundingSet=` (empty) | drop every Linux capability |
| `MemoryMax=`, `CPUQuota=` | hard resource limits through cgroups (Lab 16) |

Add restrictions a few at a time and restart after each batch. When one breaks the service, the journal says which path or call was denied. `StateDirectory=` is why `ProtectSystem=strict` did not break the write to `beats.log`.

### Exercises 4

1. Change the heartbeat interval to 10 seconds using a drop-in, and confirm from the journal.
2. Make the service write a line `"starting up"` to the journal **before** the main program starts, using `ExecStartPre=` and `/usr/bin/logger`.
3. Change `Restart=on-failure` to `Restart=always`, then `kill -TERM` the main PID directly (not through systemctl). What happens, and how does that differ from `systemctl stop`?
4. Move `HEARTBEAT_INTERVAL` out of the unit into `/etc/lab-heartbeat.env` and load it with `EnvironmentFile=`. Why is this better for secrets?
5. Reproduce the SELinux failure: copy the script to `~/labwork/09/hb`, create `lab-broken.service` with `ExecStart=` pointing at it and `User=` set to your own account, start it, and diagnose it from `systemctl status` and `sudo ausearch -m AVC -ts recent`. (Your home directory's mode may block `labsvc` too, which is why this one runs as you.)

<details><summary>Solutions</summary>

```bash
# 1
sudo systemctl edit lab-heartbeat      # add under [Service]:  Environment=HEARTBEAT_INTERVAL=10
sudo systemctl restart lab-heartbeat; sudo journalctl -u lab-heartbeat -n 2 --no-pager

# 2  (in the drop-in)
#   [Service]
#   ExecStartPre=/usr/bin/logger -t lab-heartbeat "starting up"

# 3
# With Restart=always the direct SIGTERM leads to a clean exit 0, and systemd starts it again
# 3 seconds later. systemctl stop records that the administrator wants it stopped, so no restart.

# 4
echo 'HEARTBEAT_INTERVAL=7' | sudo tee /etc/lab-heartbeat.env > /dev/null
sudo chmod 600 /etc/lab-heartbeat.env
#   [Service]
#   EnvironmentFile=/etc/lab-heartbeat.env
# The unit file is world-readable and shown by "systemctl cat". A root-owned 600 file is not,
# and it can be managed separately (for example templated from Ansible Vault).

# 5
cp /usr/local/bin/lab-heartbeat ~/labwork/09/hb
sudo tee /etc/systemd/system/lab-broken.service > /dev/null <<EOF
[Service]
User=$USER
ExecStart=$HOME/labwork/09/hb
EOF
sudo systemctl daemon-reload; sudo systemctl start lab-broken
systemctl status lab-broken | head -8        # code=exited, status=203/EXEC
sudo journalctl -u lab-broken -n 3 --no-pager        # "Failed to execute ...: Permission denied"
sudo ausearch -m AVC -ts recent | tail -3            # denied { execute } ... tcontext=...user_home_t
ls -Z ~/labwork/09/hb /usr/local/bin/lab-heartbeat   # user_home_t versus bin_t
# Fix: install the program in /usr/local/bin. Not: setenforce 0.
sudo rm /etc/systemd/system/lab-broken.service; sudo systemctl daemon-reload; sudo systemctl reset-failed
```

</details>

---

## 5. Timers: scheduled jobs

A timer unit activates a service unit on a schedule. Compared with cron you get: logs in the journal per job, `systemctl status` for the last run, dependency ordering, resource limits and sandboxing, runs caught up after downtime, and no silent failures.

A timed job needs two files with the same base name. First the job, a `oneshot` service with **no** `[Install]` section, because the timer starts it:

```bash
# If you skipped the Lab 06 challenge, replace the ExecStart command with /usr/bin/uptime
sudo tee /etc/systemd/system/lab-review.service > /dev/null <<'EOF'
[Unit]
Description=Lab: periodic server review

[Service]
Type=oneshot
ExecStart=/usr/local/bin/server-review
Nice=10
EOF
```

Then the schedule:

```bash
sudo tee /etc/systemd/system/lab-review.timer > /dev/null <<'EOF'
[Unit]
Description=Lab: run the server review every 2 minutes

[Timer]
OnCalendar=*:0/2
Persistent=true
RandomizedDelaySec=10
AccuracySec=1s

[Install]
WantedBy=timers.target
EOF

sudo systemctl daemon-reload
sudo systemctl start lab-review.service        # run the job once by hand first
sudo journalctl -u lab-review.service -n 30 --no-pager
sudo systemctl enable --now lab-review.timer   # you enable and start the TIMER, never the service
systemctl list-timers lab-review.timer
```

Wait a few minutes, then:

```bash
systemctl list-timers --all                    # NEXT, LEFT, LAST, PASSED for every timer on the system
systemctl status lab-review.timer
systemctl status lab-review.service            # result of the most recent run
sudo journalctl -u lab-review.service --since "10 minutes ago" -o short-iso | grep -c 'Finished'
```

### Schedules

`OnCalendar=` uses the form `DayOfWeek Year-Month-Day Hour:Minute:Second`, where `*` is any, `a,b` is a list, `a..b` a range and `a/n` every n starting at a.

| Expression | Meaning |
|---|---|
| `hourly`, `daily`, `weekly`, `monthly` | shorthand |
| `*-*-* 02:30:00` | every day at 02:30 |
| `Mon..Fri 08:00` | weekdays at 08:00 |
| `*:0/15` | every 15 minutes |
| `Sun 03:00` | Sundays at 03:00 |
| `*-*-01 00:00` | the first of each month |

Always check an expression before trusting it:

```bash
systemd-analyze calendar "Mon..Fri 08:00"
systemd-analyze calendar --iterations=4 "*:0/15"
systemd-analyze calendar "*-*-* 02:30:00"
```

Monotonic timers fire relative to an event instead of the clock: `OnBootSec=5min` (5 minutes after boot), `OnUnitActiveSec=1h` (an hour after the job last ran).

| Timer option | Meaning |
|---|---|
| `Persistent=true` | if the machine was off when a run was due, run it at next boot |
| `RandomizedDelaySec=` | spread start times, so that many machines do not all fire together |
| `AccuracySec=` | how much systemd may delay to batch wake-ups (default 1 minute) |
| `Unit=` | activate a unit with a different name |

### The cron equivalent

Cron is still installed and still fine for simple jobs. The same schedule as a system crontab entry:

```bash
cat /etc/crontab                               # format reminder is in the file's comments
sudo ls /etc/cron.d /etc/cron.daily
crontab -l 2>/dev/null                         # your own crontab; edit with crontab -e
# minute hour day-of-month month day-of-week  user  command
# */2    *    *            *     *            root  /usr/local/bin/server-review >> /var/log/lab-review.log 2>&1
```

| | cron | systemd timer |
|---|---|---|
| Set-up | one line | two files |
| Output | mailed or lost unless redirected | journal, per unit |
| Missed runs | skipped (anacron helps for daily jobs) | `Persistent=true` |
| Overlap | starts a second copy | never: the service is still active |
| Last run status | none | `systemctl status`, exit code, duration |
| Dependencies, limits, sandbox | no | everything from section 4 |
| Environment | minimal, a frequent source of surprises | explicit in the unit |

For a nightly data load that depends on a database being up, a timer with `After=`/`Requires=` on the database's unit, `Persistent=true`, and a journal you can query is the more robust choice.

A failed timer job is visible in `systemctl --failed`, which your review script already reports.

### Exercises 5

1. Change the timer to run at 20 past every hour. Verify with `systemd-analyze calendar` and `list-timers`.
2. Write the `OnCalendar` expression for "02:15 every night" and for "every 6 hours". Check both.
3. Add `OnFailure=` handling: create a tiny `lab-alert@.service` template that logs a message with `logger`, and make `lab-review.service` trigger it when it fails. Force a failure to test.
4. How long did the last run of `lab-review.service` take? (Hint: `systemctl show` properties with `Timestamp` in the name, or the "Consumed" line in the journal.)

<details><summary>Solutions</summary>

```bash
# 1
sudo systemctl edit lab-review.timer
#   [Timer]
#   OnCalendar=
#   OnCalendar=*:20
systemd-analyze calendar "*:20"; systemctl list-timers lab-review.timer

# 2
systemd-analyze calendar "*-*-* 02:15:00"
systemd-analyze calendar "0/6:00"           # 00:00, 06:00, 12:00, 18:00

# 3
sudo tee /etc/systemd/system/lab-alert@.service > /dev/null <<'EOF'
[Unit]
Description=Lab alert for %i

[Service]
Type=oneshot
ExecStart=/usr/bin/logger -t lab-alert -p user.err "unit %i FAILED"
EOF
sudo systemctl edit lab-review.service
#   [Unit]
#   OnFailure=lab-alert@%n.service
#   [Service]
#   ExecStart=
#   ExecStart=/usr/bin/false
sudo systemctl start lab-review.service; sudo journalctl -t lab-alert -n 2 --no-pager
sudo systemctl revert lab-review.service; sudo systemctl reset-failed

# 4
systemctl show lab-review.service -p ExecMainStartTimestamp -p ExecMainExitTimestamp
sudo journalctl -u lab-review.service -n 3 --no-pager     # "Consumed 120ms CPU time."
```

</details>

---

## 6. When a unit will not start

A fixed routine finds nearly every problem.

```bash
systemctl status UNIT                    # 1. state, exit code, last log lines
sudo journalctl -u UNIT -b -e            # 2. the full story
sudo journalctl -xeu UNIT                # 2b. with explanations
systemctl cat UNIT                       # 3. what is systemd actually running? (drop-ins included)
sudo systemd-analyze verify /etc/systemd/system/UNIT.service      # 4. syntax
sudo -u SVCUSER /path/to/ExecStart       # 5. does it run by hand as that user?
sudo ausearch -m AVC -ts recent          # 6. SELinux
```

Exit statuses systemd adds on top of the program's own:

| Status | Meaning | Usual cause |
|---|---|---|
| `203/EXEC` | could not execute the command | wrong path, not executable, missing interpreter line, CRLF line endings, SELinux |
| `217/USER` | could not switch user | the `User=` account does not exist |
| `200/CHDIR` | could not enter `WorkingDirectory=` | missing directory or no permission |
| `226/NAMESPACE` | could not set up sandboxing | a path named in `ReadWritePaths=` and similar does not exist |
| `status=1/FAILURE` and other small numbers | the program started and exited with that code | read its own log output |
| `code=killed, signal=KILL` | SIGKILL | OOM killer (`journalctl -k \| grep -i oom`), or a stop timeout |
| `start-limit-hit` | crashed repeatedly and systemd stopped trying | fix the cause, then `reset-failed` |
| `timeout` | did not become ready within `TimeoutStartSec` | wrong `Type=` (commonly `forking` on a program that stays in the foreground) |

### Challenge: fix four broken units

Create these, then get each one working. Diagnose from systemd's output before reading the unit. Three of them fail visibly. The fourth reports `active (running)` and is broken anyway, so check the journal of every one.

```bash
sudo tee /etc/systemd/system/lab-fix1.service > /dev/null <<'EOF'
[Service]
ExecStart=/usr/local/bin/lab-heartbeet
EOF

sudo tee /etc/systemd/system/lab-fix2.service > /dev/null <<'EOF'
[Service]
User=labservice
ExecStart=/usr/local/bin/lab-heartbeat
EOF

sudo tee /etc/systemd/system/lab-fix3.service > /dev/null <<'EOF'
[Service]
User=labsvc
ExecStart=/usr/local/bin/lab-heartbeat
ProtectSystem=strict
Environment=STATE_DIRECTORY=/var/lib/lab-fix3
EOF

sudo tee /etc/systemd/system/lab-fix4.service > /dev/null <<'EOF'
[Service]
Type=forking
User=labsvc
ExecStart=/usr/local/bin/lab-heartbeat
TimeoutStartSec=10
EOF

sudo systemctl daemon-reload
for n in 1 2 3 4; do sudo systemctl start lab-fix$n --no-block; done
sleep 12; systemctl --failed
```

<details><summary>Diagnoses</summary>

| Unit | Symptom | Cause | Fix |
|---|---|---|---|
| `lab-fix1` | `status=203/EXEC`, "No such file or directory" | typo in the path: `heartbeet` | correct `ExecStart=` |
| `lab-fix2` | `status=217/USER` | account `labservice` does not exist | `User=labsvc` |
| `lab-fix3` | `active (running)`, not in `--failed`; the journal shows `beats.log: No such file or directory` on every beat | the state directory was never created, and under `ProtectSystem=strict` the service could not create it anyway | replace the `Environment=` line with `StateDirectory=lab-fix3` |
| `lab-fix4` | sits in `activating` for 10 s, then `timeout`, though the program was running fine | `Type=forking` waits for the main process to exit after daemonising; this program stays in the foreground | `Type=simple` (or delete the line) |

After each edit: `sudo systemctl daemon-reload && sudo systemctl restart lab-fixN && systemctl status lab-fixN`.

`lab-fix3` is the one to remember. The script carries on when its write fails, so systemd sees a live process and reports success. "Active" means the process exists. Whether it is doing its job is something only its logs, or a health check, can tell you.

</details>

## Check yourself

<details><summary>What is the difference between <code>systemctl start</code> and <code>systemctl enable</code>?</summary>
<code>start</code> runs the unit now and has no effect on the next boot. <code>enable</code> creates the symlink that starts it at boot and has no effect now. <code>enable --now</code> does both.
</details>

<details><summary>You edited a unit file and restarted the service, but nothing changed. What did you forget?</summary>
<code>systemctl daemon-reload</code>. systemd was still using the copy of the unit it had in memory.
</details>

<details><summary>What is the difference between <code>After=</code> and <code>Requires=</code>?</summary>
<code>After=</code> is ordering only: if both are starting, this one goes second. It does not cause the other to start. <code>Requires=</code> is a dependency: it pulls the other in and fails if that fails, but says nothing about order. Most real dependencies need both.
</details>

<details><summary>How do you change one setting of a packaged service so that the change survives updates?</summary>
<code>systemctl edit unit</code>, which creates a drop-in under <code>/etc/systemd/system/unit.d/</code>. Files in <code>/etc</code> take precedence over <code>/usr/lib</code> and are not touched by package updates.
</details>

<details><summary>A service shows <code>status=203/EXEC</code>, but the file exists and is mode 755. What are two things to check on Rocky?</summary>
The first line of the script (a missing or wrong <code>#!</code> interpreter, or Windows line endings), and SELinux: the file's label with <code>ls -Z</code> and recent denials with <code>ausearch -m AVC -ts recent</code>.
</details>

<details><summary>Why is a timer's service unit <code>Type=oneshot</code> with no <code>[Install]</code> section?</summary>
The job runs to completion and exits, which is what oneshot describes, so systemd records success or failure when it ends. It has no <code>[Install]</code> because it should never start at boot by itself; the timer activates it.
</details>

<details><summary>Give three advantages of a systemd timer over a cron entry.</summary>
Output and exit status are captured per job in the journal; missed runs can be caught up with <code>Persistent=true</code>; jobs never overlap; and the job can use dependencies, resource limits and sandboxing like any other unit.
</details>

## Cleanup

```bash
sudo systemctl disable --now lab-review.timer lab-heartbeat lab-sleeper 2>/dev/null
sudo systemctl stop 'lab-*' 2>/dev/null
sudo rm -f  /etc/systemd/system/lab-*.service /etc/systemd/system/lab-*.timer
sudo rm -rf /etc/systemd/system/lab-*.d
sudo rm -f  /usr/local/bin/lab-heartbeat /etc/lab-heartbeat.env
sudo rm -rf /var/lib/lab-heartbeat /var/lib/lab-fix3
sudo systemctl daemon-reload
sudo systemctl reset-failed
sudo userdel labsvc 2>/dev/null
rm -rf ~/labwork/09

systemctl list-units 'lab-*' --all --no-legend      # expect nothing
systemctl --failed                                  # expect 0 loaded units
```

Keep `/usr/local/bin/server-review`. If you want the periodic review for real, recreate `lab-review.service` and `.timer` under a non-lab name with a sensible schedule such as `OnCalendar=daily`, and add them to your Ansible playbook so the tower can be rebuilt with them.
