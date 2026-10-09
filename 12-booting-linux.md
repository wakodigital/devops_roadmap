# Lab 12 — Booting Linux

**Roadmap topics:** Logs · Boot Loaders
**Time:** about 2.5 hours
**Touches system state:** almost entirely read-only. Section 4 creates and removes one logrotate config and one test log file. **No boot loader setting is changed.** A headless machine on Wi-Fi that fails to boot can only be fixed with a monitor and keyboard, so on the tower you will inspect the boot loader thoroughly and learn the change procedures as text. An optional, clearly marked exercise reboots the machine.

## What you will be able to do

- Describe each stage between the power button and a login prompt, and say where each stage's configuration and evidence live.
- Read the tower's boot configuration: firmware mode, boot entries, kernels, kernel command line.
- Explain how a boot parameter is changed permanently and for one boot, and how a machine is rescued when it will not boot.
- Find out how long boot took and what slowed it.
- Read kernel and boot logs, filter by priority, and compare this boot with the last one.
- Explain where each log on the system comes from and how rotation works.

## Setup

```bash
mkdir -p ~/labwork/12 && cd ~/labwork/12
```

---

## 1. The boot sequence

| # | Stage | What it does | Where it lives | Evidence |
|---|---|---|---|---|
| 1 | **Firmware** (UEFI, or legacy BIOS) | powers on and tests hardware, then finds something bootable | the motherboard's flash chip | `efibootmgr -v`, firmware setup screen |
| 2 | **Boot loader** (GRUB 2, started via `shim` under Secure Boot) | shows the menu, loads the chosen kernel and initramfs into memory, passes the kernel command line | UEFI: the EFI System Partition, `/boot/efi`. BIOS: the first sectors of the disk. Config in `/boot/grub2` and `/boot/loader/entries` | `grubby --info=ALL`, `/proc/cmdline` |
| 3 | **Kernel** | initialises CPUs, memory and built-in drivers, unpacks the initramfs | `/boot/vmlinuz-VERSION` | `dmesg`, `journalctl -k` |
| 4 | **initramfs** | a small temporary root filesystem in RAM with just enough drivers and tools (storage, LVM, encryption) to find and mount the real root filesystem | `/boot/initramfs-VERSION.img`, built by `dracut` | `lsinitrd`, early lines of `journalctl -b` |
| 5 | **systemd** (PID 1) | mounts the filesystems in fstab, starts services in dependency order until the default target is reached | `/usr/lib/systemd/systemd`, units in `/usr/lib/systemd/system` and `/etc/systemd/system` | `systemd-analyze`, `journalctl -b` |
| 6 | **Login** | `getty` on consoles, `sshd` on the network | | `/var/log/secure` |

Knowing the stages is what turns "it won't boot" into a diagnosis. Where it stops tells you which stage failed:

| What you see on the console | Failed stage | Usual cause |
|---|---|---|
| nothing, or "no bootable device" | firmware | boot order, dead disk, missing EFI entry |
| `grub>` or `grub rescue>` prompt | boot loader | GRUB cannot find its config or `/boot` |
| kernel panic: "unable to mount root fs" | kernel / initramfs | initramfs lacks the storage driver, or wrong `root=` |
| `dracut:/#` emergency shell | initramfs | root device not found: wrong UUID, missing LVM volume |
| "You are in emergency mode… Give root password for maintenance" | systemd, early | **an fstab entry that cannot be mounted** (Lab 11), or filesystem corruption |
| boots, but a service is down | systemd, late | a failed unit: Lab 09 |

### Walk each stage on the tower

```bash
# 1. firmware
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "legacy BIOS"
sudo efibootmgr -v 2>/dev/null              # UEFI boot entries and their order
mokutil --sb-state 2>/dev/null              # is Secure Boot on?
sudo dmidecode -s bios-version; sudo dmidecode -s system-product-name

# 2. boot loader
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS | grep -E 'boot|NAME'
sudo ls /boot /boot/efi/EFI 2>/dev/null
cat /proc/cmdline                           # the command line GRUB passed to the running kernel

# 3. kernel
uname -r
ls -lh /boot/vmlinuz-*
sudo dmesg | head -20

# 4. initramfs
ls -lh /boot/initramfs-*
sudo lsinitrd | head -30                    # contents of the current initramfs: modules, dracut hooks
sudo lsinitrd | grep -ciE 'lvm|nvme|xfs'

# 5. systemd
systemctl get-default
systemd-analyze                             # time spent in firmware, loader, kernel, initrd, userspace
```

---

## 2. Boot loaders

A boot loader's job is small and critical: present a choice, load a kernel and an initramfs, hand over a command line.

| Boot loader | Where you meet it |
|---|---|
| **GRUB 2** | nearly every x86 server distribution, including Rocky. BIOS and UEFI |
| **systemd-boot** | UEFI only, minimal. Default on some desktop distributions; common on Arch |
| **shim** | a small signed first-stage loader that makes Secure Boot work, then starts GRUB |
| **U-Boot** | ARM boards and embedded devices |
| **SYSLINUX / ISOLINUX** | install media and live USB images |
| **LILO**, GRUB Legacy | historical |

### GRUB 2 on Rocky 9: the files

| File | Role | Edit? |
|---|---|---|
| `/etc/default/grub` | your settings: timeout, default entry, extra kernel arguments | **yes** |
| `/etc/grub.d/` | scripts that generate the menu | rarely |
| `/boot/grub2/grub.cfg` | the generated configuration GRUB reads | **never by hand** |
| `/boot/loader/entries/*.conf` | one small file per installed kernel (the Boot Loader Specification, "BLS"). This is where each entry's kernel, initramfs and arguments are | through `grubby` |
| `/boot/grub2/grubenv` | saved state: which entry is the default | through `grubby` / `grub2-editenv` |
| `/boot/efi/EFI/rocky/` | on UEFI: `shimx64.efi`, `grubx64.efi`, and a stub `grub.cfg` that points at `/boot/grub2/grub.cfg` | no |

```bash
cat /etc/default/grub
sudo ls -l /boot/loader/entries/
sudo cat /boot/loader/entries/*"$(uname -r)".conf
sudo grub2-editenv list
sudo head -20 /boot/grub2/grub.cfg
```

A BLS entry:

```
title Rocky Linux (5.14.0-503.14.1.el9_5.x86_64) 9.5 (Blue Onyx)
version 5.14.0-503.14.1.el9_5.x86_64
linux /vmlinuz-5.14.0-503.14.1.el9_5.x86_64
initrd /initramfs-5.14.0-503.14.1.el9_5.x86_64.img
options root=/dev/mapper/rl-root ro crashkernel=1G-4G:192M,4G-64G:256M,64G-:512M resume=/dev/mapper/rl-swap rd.lvm.lv=rl/root rd.lvm.lv=rl/swap rhgb quiet
```

Paths are relative to the `/boot` filesystem. `options` is the kernel command line.

### grubby: the supported interface

```bash
sudo grubby --default-kernel                # which kernel boots by default
sudo grubby --default-index
sudo grubby --info=ALL                      # every entry: index, kernel, args, root, initrd, title
sudo grubby --info=DEFAULT
rpm -q kernel                               # installed kernels (up to installonly_limit, Lab 10)
```

### Kernel command-line parameters

```bash
cat /proc/cmdline
```

| Parameter | Meaning |
|---|---|
| `root=…` | the root filesystem to mount |
| `ro` | mount it read-only at first; systemd remounts it read-write after checks |
| `rd.lvm.lv=vg/lv` | LVM volumes the initramfs must activate |
| `resume=…` | the swap device to resume from hibernation |
| `crashkernel=…` | memory reserved for the kdump crash-capture kernel |
| `rhgb` | Red Hat graphical boot: the splash screen |
| `quiet` | hide most boot messages |
| `console=ttyS0,115200` | send the console to a serial port. Essential on VMs and cloud instances |
| `systemd.unit=rescue.target` | boot to single-user rescue mode |
| `systemd.unit=emergency.target` | boot to the most minimal shell, root filesystem read-only |
| `rd.break` | stop inside the initramfs, before the real root is mounted |
| `init=/bin/bash` | run a shell in place of systemd |
| `enforcing=0` | start SELinux in permissive mode |
| `nomodeset` | do not load graphics drivers: for display problems |
| `mitigations=off`, `intel_iommu=on`, `isolcpus=…` | performance and virtualisation tuning you will see in guides |

`man 7 bootparam`, `man 7 kernel-command-line` and `man 7 dracut.cmdline` document them.

### Changing boot settings (read; do not run on the tower)

**Permanently, for all kernels.** Use `grubby`. It edits the BLS entries directly.

```bash
# sudo grubby --update-kernel=ALL --args="console=ttyS0,115200"
# sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"
# sudo grubby --set-default /boot/vmlinuz-5.14.0-…          # choose the default kernel
# sudo grubby --set-default-index=1
```

**Menu behaviour** (timeout, hidden menu) is in `/etc/default/grub`, after which the config is regenerated:

```bash
# sudoedit /etc/default/grub                 # e.g. GRUB_TIMEOUT=5
# sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

On current Rocky 9, `grub2-mkconfig` alone does **not** push `GRUB_CMDLINE_LINUX` changes into existing BLS entries (it needs `--update-bls-cmdline`), which is why `grubby` is the tool for kernel arguments.

**For the next boot only.** Useful for trying a new kernel safely: if it fails, a power-cycle returns to the saved default.

```bash
# sudo grub2-reboot 1 && sudo systemctl reboot
```

**At the console, for one boot.** At the GRUB menu, press `e` on an entry, edit the `linux` line, press `Ctrl-x` to boot. Nothing is saved. This is how every rescue below starts, and it needs a keyboard and screen on the machine.

**The safe way to make any boot change**

1. Copy what you are changing: `sudo cp -a /etc/default/grub /etc/default/grub.bak`, and note `sudo grubby --info=DEFAULT`.
2. Make one change.
3. Keep at least one older, known-good kernel entry untouched.
4. Have console access arranged **before** you reboot.
5. Reboot once, on purpose, while you are there to deal with it.

### Rescue procedures (knowledge for the day you need it)

All of these start at the GRUB menu with `e`.

| Problem | Do |
|---|---|
| New kernel will not boot | select the previous kernel from the menu. Then `grubby --set-default` to it, and investigate |
| A service or mount hangs the boot | append `systemd.unit=rescue.target`. Root password required. Fix, then `systemctl default` or reboot |
| fstab error, emergency mode | at the emergency prompt: `mount -o remount,rw /`, fix `/etc/fstab`, `systemctl daemon-reload`, `systemctl default` |
| Forgotten root password | append `rd.break`; then `mount -o remount,rw /sysroot`, `chroot /sysroot`, `passwd root`, `touch /.autorelabel`, exit twice. The relabel is needed because `passwd` ran without SELinux policy loaded |
| GRUB itself is broken | boot the Rocky install ISO, choose Troubleshooting → Rescue, `chroot /mnt/sysroot`, then `dnf reinstall grub2-efi-x64 shim-x64` (UEFI) or `grub2-install /dev/sdX` (BIOS), and `grub2-mkconfig` |
| initramfs lacks a driver | from rescue: `dracut -f /boot/initramfs-VERSION.img VERSION` |

Anyone with physical access can do the same, which is why GRUB can be given a password (`grub2-setpassword`) and disks can be encrypted.

### Exercises 2

1. Is the tower UEFI or BIOS? Is Secure Boot on?
2. How many kernels are installed? Which is the default, and is it the one running now?
3. Take the running kernel's command line apart: explain every parameter on it.
4. What is the GRUB menu timeout? Is the menu hidden?
5. The tower is headless. Which single kernel parameter would let you watch boot messages and use the GRUB menu without a monitor, given the right cable? What hardware would it need?
6. Write (do not run) the two commands that would remove `rhgb quiet` from every kernel entry and confirm the change.

<details><summary>Solutions</summary>

```bash
# 1
[ -d /sys/firmware/efi ] && echo UEFI || echo BIOS; mokutil --sb-state 2>/dev/null

# 2
rpm -q kernel | wc -l; sudo grubby --default-kernel; uname -r

# 3
tr ' ' '\n' < /proc/cmdline         # one per line; look each up in the table above
                                    # BOOT_IMAGE= is added by GRUB itself: the kernel file it loaded

# 4
grep -E 'GRUB_TIMEOUT|GRUB_TIMEOUT_STYLE' /etc/default/grub
sudo grub2-editenv list             # menu_auto_hide=1 hides the menu after a successful boot

# 5
# console=ttyS0,115200 (plus GRUB_TERMINAL="serial console" for the menu). It needs a serial port
# on the motherboard and a null-modem or USB-serial cable to another computer. Servers get the same
# result from a BMC (IPMI, iDRAC, iLO); a network KVM does it for a desktop tower.

# 6
# sudo grubby --update-kernel=ALL --remove-args="rhgb quiet"
# sudo grubby --info=ALL | grep args
```

</details>

---

## 3. Boot and kernel logs

### How long did boot take, and why

```bash
systemd-analyze                             # totals per stage
systemd-analyze blame | head -15            # units by time spent starting, slowest first
systemd-analyze critical-chain              # the chain of units that determined when boot finished
systemd-analyze critical-chain docker.service
systemd-analyze plot > boot.svg             # a timeline chart; copy it to your laptop and open in a browser
```

`blame` lists durations, but units start in parallel, so a slow unit only delays boot if something waits for it. `critical-chain` shows what was actually waited on: `@` is when a unit became active and `+` is how long it took. A common culprit is `NetworkManager-wait-online.service`, more so over Wi-Fi.

### The kernel ring buffer

The kernel logs into a fixed-size buffer in memory, readable with `dmesg`. It covers hardware detection, drivers, filesystems, the OOM killer, and anything the kernel wants to tell you.

```bash
sudo dmesg | less                           # timestamps are seconds since boot
sudo dmesg -T | tail -20                    # human-readable wall-clock time
sudo dmesg -H                               # relative times, paged, coloured
sudo dmesg -l err,crit,alert,emerg          # only serious messages
sudo dmesg -l warn | tail
sudo dmesg -T | grep -iE 'error|fail|warn' | tail -20
sudo dmesg -w                               # follow: plug in a USB stick and watch (Ctrl-C to stop)
sudo dmesg -T | grep -iE 'nvme|sd[a-z]|ata[0-9]' | head -20        # disks being detected
sudo dmesg -T | grep -iE 'wlan|wlp|iwl|firmware' | head            # Wi-Fi adapter
sudo dmesg -T | grep -i 'out of memory'
```

The buffer is a ring: on a long-running or noisy machine the oldest entries are overwritten. The journal keeps a persistent copy:

```bash
sudo journalctl -k                          # kernel messages, this boot
sudo journalctl -k -b -1                    # kernel messages from the PREVIOUS boot: what dmesg cannot show
sudo journalctl -k -p err -b
```

### The journal, by boot

```bash
sudo journalctl --list-boots                # index, boot ID, first and last entry of each recorded boot
sudo journalctl -b                          # this boot, from the first line
sudo journalctl -b -1                       # the previous boot
sudo journalctl -b -1 -e                    # the END of the previous boot: how did it shut down?
sudo journalctl -b -p err                   # this boot's errors: the standard first look
sudo journalctl -b -p warning | wc -l
sudo journalctl -b -1 -p err --no-pager | tail -20
sudo journalctl -b -u systemd-fsck-root -u systemd-remount-fs --no-pager
sudo cat /var/log/boot.log | tail -30       # the [  OK  ] / [FAILED] lines that scroll past on the console
```

Whether boots before the current one are available depends on the journal being persistent (`/var/log/journal` exists) and how much space it is allowed (Lab 09).

### Log levels

The kernel, syslog and the journal share one severity scale. `-p N` or `-l name` shows that level and everything more severe.

| # | Name | Meaning |
|---|---|---|
| 0 | `emerg` | the system is unusable |
| 1 | `alert` | act immediately |
| 2 | `crit` | critical condition: hardware failure |
| 3 | `err` | an error |
| 4 | `warning` | a warning |
| 5 | `notice` | normal but significant |
| 6 | `info` | informational |
| 7 | `debug` | debugging detail |

```bash
for p in emerg alert crit err warning notice info; do
  printf '%-8s %s\n' "$p" "$(sudo journalctl -b -p "$p..$p" --no-pager -q | wc -l)"
done
```

Every healthy machine logs some errors at boot: firmware quirks, optional hardware that is absent, ACPI complaints. Learn what **your** machine's normal looks like now, so that a new line stands out later:

```bash
sudo journalctl -b -p err --no-pager -o cat | sort | uniq -c | sort -rn > ~/labwork/12/baseline-errors.txt
wc -l ~/labwork/12/baseline-errors.txt; head ~/labwork/12/baseline-errors.txt
```

### Was the last shutdown clean?

```bash
last -x | grep -E 'reboot|shutdown|crash' | head
sudo journalctl -b -1 -e --no-pager | tail -15
```

A clean shutdown ends with systemd stopping units, unmounting filesystems and "Journal stopped". A previous boot's log that simply ends in the middle of normal activity means power loss, a hard reset or a kernel crash.

### Exercises 3

1. How long did the last boot take in total, and how much of that was firmware?
2. Which three units took longest to start? Was any of them on the critical chain?
3. How many boots does the journal hold? When was the oldest?
4. List the kernel messages of level `err` or worse from this boot. For each, decide: hardware, driver, or harmless firmware noise?
5. Find the kernel line where the root filesystem was mounted, and the line where systemd announced itself.
6. Did the previous boot end cleanly?

<details><summary>Solutions</summary>

```bash
systemd-analyze                                                   # 1
systemd-analyze blame | head -3; systemd-analyze critical-chain   # 2
sudo journalctl --list-boots | wc -l; sudo journalctl --list-boots | head -1     # 3
sudo journalctl -k -b -p err --no-pager                           # 4
sudo journalctl -b --no-pager | grep -m1 -iE 'xfs.*mounting|mounted filesystem'  # 5
sudo journalctl -b --no-pager | grep -m1 'systemd\[1\]: systemd .* running in system mode'
sudo journalctl -b -1 -e --no-pager | tail -5                     # 6
```

</details>

---

## 4. The logging system as a whole

Two systems run side by side on Rocky and receive the same events.

```
  kernel ──────────────┐
  services' stdout ────┤
  syslog() calls ──────┼──▶ systemd-journald ──▶ /var/log/journal/   (binary, indexed: journalctl)
  logger command ──────┘            │
                                    └──▶ rsyslog ──▶ /var/log/messages, secure, cron, maillog   (plain text)
                                                          │
                                                          └── logrotate: rotate, compress, delete
```

### A tour of /var/log

```bash
sudo ls -lt /var/log | head -25
```

| File | Contents | Written by |
|---|---|---|
| `messages` | general system log: most things not routed elsewhere | rsyslog |
| `secure` | authentication, sudo, sshd (Lab 06) | rsyslog |
| `cron` | cron job starts | rsyslog |
| `maillog` | mail subsystem | rsyslog |
| `boot.log` | console output of the boot | plymouth |
| `dnf.log`, `dnf.rpm.log` | package operations (Lab 10) | dnf |
| `audit/audit.log` | the kernel audit system, including SELinux denials. Read with `ausearch`, `aureport` | auditd |
| `firewalld` | firewall daemon messages | firewalld |
| `wtmp`, `btmp`, `lastlog` | binary login records: read with `last`, `lastb`, `lastlog` | login programs |
| `journal/` | the journal | journald |
| `sa/` | system activity history: read with `sar` (sysstat) | sysstat |
| `anaconda/` | the log of the original OS installation | installer |

Applications that manage their own logs write to `/var/log/NAME/`. Containers do not appear here at all: Docker keeps each container's output under `/var/lib/docker/containers/` and shows it with `docker logs`.

### rsyslog: routing

```bash
grep -vE '^\s*(#|$)' /etc/rsyslog.conf
ls /etc/rsyslog.d/
```

Rules have the form `facility.priority   destination`:

```
authpriv.*                      /var/log/secure
cron.*                          /var/log/cron
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
*.emerg                         :omusrmsg:*
```

The **facility** is the kind of source (`auth`, `authpriv`, `cron`, `daemon`, `kern`, `mail`, `user`, `local0`–`local7`). The **priority** is the level from the table above, meaning "this and worse". A destination of `@host:514` (UDP) or `@@host:514` (TCP) forwards to a remote log server, which is how logs are centralised and kept out of reach of someone who compromises the machine.

```bash
logger -p local0.notice -t lab12 "routing test"
sudo grep lab12 /var/log/messages
sudo journalctl -t lab12 --no-pager
```

### logrotate: keeping logs from filling the disk

`logrotate` runs daily from a systemd timer. For each log it is configured for, it renames the current file, optionally compresses old ones, deletes those beyond a retention count, and tells the application to reopen its log.

```bash
systemctl list-timers logrotate.timer
cat /etc/logrotate.conf
ls /etc/logrotate.d/
cat /etc/logrotate.d/rsyslog
sudo ls -l /var/log/messages* /var/log/secure*
```

Write a rotation policy for a log of your own, test it, and remove it.

```bash
sudo mkdir -p /var/log/lab12
for i in $(seq 1 2000); do echo "$(date -Is) line $i of the lab application log"; done | sudo tee /var/log/lab12/app.log > /dev/null

sudo tee /etc/logrotate.d/lab12 > /dev/null <<'EOF'
/var/log/lab12/*.log {
    daily
    rotate 4
    size 10k
    compress
    delaycompress
    missingok
    notifempty
    create 0640 root root
    dateext
}
EOF

sudo logrotate -d /etc/logrotate.d/lab12          # -d: debug. Explains what it WOULD do and changes nothing
sudo logrotate -f /etc/logrotate.d/lab12          # -f: force a rotation now
sudo ls -l /var/log/lab12/
echo "after rotation" | sudo tee -a /var/log/lab12/app.log > /dev/null
sudo logrotate -f /etc/logrotate.d/lab12
sudo ls -l /var/log/lab12/                        # with dateext, a second rotation on the same day is skipped
```

| Directive | Meaning |
|---|---|
| `daily` / `weekly` / `monthly` | how often |
| `rotate 4` | keep four old files |
| `size 10k` / `maxsize` / `minsize` | rotate by size, instead of or as well as by time |
| `compress`, `delaycompress` | gzip old logs; leave the newest old one uncompressed for a cycle |
| `missingok` | no error if the log is absent |
| `notifempty` | do not rotate an empty file |
| `create MODE OWNER GROUP` | create a fresh log after rotating |
| `copytruncate` | copy then empty the file in place, for programs that cannot reopen their log (a few lines may be lost) |
| `dateext` | name old logs by date, not `.1`, `.2` |
| `postrotate … endscript` | a command to run afterwards, typically a reload or `kill -HUP` so the daemon reopens its log |
| `sharedscripts` | run the script once for all matched files |

The reason for `postrotate` is the open-file behaviour from Lab 04: after a rename, a daemon keeps writing to the old inode until it reopens the file.

Programs that log to stdout under systemd need none of this: journald does retention itself.

```bash
sudo rm -rf /var/log/lab12 /etc/logrotate.d/lab12     # clean up now
```

### Exercises 4

1. Which is the most recently written file in `/var/log`? Which is the largest?
2. From `/etc/rsyslog.conf`: where do messages with facility `authpriv` go? Send a test message there with `logger` and find it.
3. How many rotated copies of `/var/log/messages` are kept, and how often is it rotated? Where is that configured?
4. When did logrotate last run, and when will it next run?
5. Write a logrotate stanza for `/var/log/myapp/*.log`: weekly, keep 8, compressed, and send `SIGHUP` to the process whose PID is in `/run/myapp.pid` afterwards. Validate the syntax with `logrotate -d`, then delete it.

<details><summary>Solutions</summary>

```bash
# 1
sudo ls -t /var/log | head -1; sudo du -ah /var/log 2>/dev/null | sort -h | tail -3

# 2
grep authpriv /etc/rsyslog.conf
logger -p authpriv.notice -t lab12 "authpriv test"; sudo grep lab12 /var/log/secure

# 3
cat /etc/logrotate.d/rsyslog; grep -E '^(weekly|daily|rotate)' /etc/logrotate.conf
# The rsyslog stanza inherits "weekly" and "rotate 4" from /etc/logrotate.conf unless it overrides them.

# 4
systemctl list-timers logrotate.timer; sudo journalctl -u logrotate -n 5 --no-pager

# 5
sudo tee /etc/logrotate.d/myapp-test > /dev/null <<'EOF'
/var/log/myapp/*.log {
    weekly
    rotate 8
    compress
    missingok
    notifempty
    sharedscripts
    postrotate
        /bin/kill -HUP "$(cat /run/myapp.pid 2>/dev/null)" 2>/dev/null || true
    endscript
}
EOF
sudo logrotate -d /etc/logrotate.d/myapp-test; sudo rm /etc/logrotate.d/myapp-test
```

</details>

---

## 5. Targets, shutdown and reboot

A **target** groups units and marks a point in boot. Targets replaced the old numbered runlevels.

| Target | Old runlevel | State |
|---|---|---|
| `poweroff.target` | 0 | off |
| `rescue.target` | 1 | single user: local filesystems mounted, no network services |
| `multi-user.target` | 3 | full system, text mode. **The default on a server** |
| `graphical.target` | 5 | multi-user plus a display manager |
| `reboot.target` | 6 | reboot |
| `emergency.target` | | the bare minimum: root mounted read-only, a shell |

```bash
systemctl get-default
systemctl list-units --type=target
systemctl list-dependencies multi-user.target | head -30
# sudo systemctl set-default multi-user.target      # change what the machine boots into
# sudo systemctl isolate rescue.target              # switch NOW. Over SSH this cuts you off.
```

```bash
# sudo systemctl reboot                    # the normal way
# sudo systemctl poweroff
# sudo shutdown -r +10 "Rebooting for kernel update"     # in 10 minutes, with a warning to logged-in users
# sudo shutdown -c                         # cancel a pending shutdown
# sudo shutdown -r 02:00                   # at a time
```

### Optional exercise: a deliberate reboot

Reading boot logs is more useful when you have just watched the boot they describe. This restarts everything the tower runs, so choose your moment, and skip it if you cannot get to the machine physically should it not return.

Pre-flight. Every line should be clean before you go on:

```bash
sudo findmnt --verify                      # fstab is sound
systemctl --failed                         # nothing already broken
sudo sshd -t                               # sshd config parses
systemctl is-enabled sshd docker NetworkManager
systemctl is-enabled cloudflared 2>/dev/null
grep -c lab /etc/fstab                     # 0: no leftover lab mounts
sudo grubby --default-kernel; uname -r     # will it boot the kernel it is running now?
docker ps --format '{{.Names}}' | sort > ~/labwork/12/before.txt
date
```

```bash
sudo systemctl reboot
```

Reconnect when it answers again (`ping`, then `ssh`), then:

```bash
uptime; systemd-analyze
systemctl --failed
sudo journalctl -b -p err --no-pager -o cat | sort -u > ~/labwork/12/after-errors.txt
sudo journalctl -b -1 -e --no-pager | tail -8          # the end of the previous boot: a clean shutdown
diff ~/labwork/12/before.txt <(docker ps --format '{{.Names}}' | sort) && echo "same containers running"
```

Compare `after-errors.txt` with the messages in `baseline-errors.txt`: any error that is new since the baseline is worth reading.

If a container did not come back, check its restart policy: `docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' NAME`. A service that only runs because someone started it by hand is the most common thing a reboot exposes.

## Challenge: the boot report

Write `~/labwork/12/boot-report.sh`, which prints for the **current** boot:

- firmware type and Secure Boot state
- running kernel, default kernel, and whether they match
- the kernel command line
- boot time, total and by stage
- the five slowest units
- counts of journal messages at `err` and at `warning`
- any failed units
- whether the previous boot shut down cleanly (look for "Journal stopped" or a `systemd-shutdown` line in its last entries)

<details><summary>One solution</summary>

```bash
cat > ~/labwork/12/boot-report.sh <<'EOF'
#!/usr/bin/env bash
# Boot report for the current boot. Run with sudo.
h() { printf '\n== %s ==\n' "$1"; }

h "Firmware"
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "legacy BIOS"
mokutil --sb-state 2>/dev/null || echo "Secure Boot state: unavailable"

h "Kernel"
running="/boot/vmlinuz-$(uname -r)"; default="$(grubby --default-kernel)"
echo "running: $running"; echo "default: $default"
[ "$running" = "$default" ] && echo "match: yes" || echo "match: NO (a reboot would change kernel)"
echo "cmdline: $(cat /proc/cmdline)"

h "Boot time"
systemd-analyze | head -1
systemd-analyze blame | head -5

h "Journal, this boot"
echo "err or worse:  $(journalctl -b -p err -q --no-pager | wc -l)"
echo "warnings:      $(journalctl -b -p warning..warning -q --no-pager | wc -l)"

h "Failed units"
systemctl --failed --no-legend --plain | awk '{print $1}' | grep . || echo "none"

h "Previous shutdown"
if journalctl -b -1 -n 40 --no-pager -q 2>/dev/null | grep -qE 'Journal stopped|systemd-shutdown|Reached target.*(Reboot|Power-Off|Shutdown)'; then
  echo "clean"
else
  echo "no shutdown messages found: unclean, or no previous boot recorded"
fi
EOF
chmod +x ~/labwork/12/boot-report.sh && sudo ~/labwork/12/boot-report.sh
```

</details>

## Check yourself

<details><summary>Name the boot stages in order.</summary>
Firmware (UEFI or BIOS), boot loader (GRUB), kernel, initramfs, systemd, login.
</details>

<details><summary>What is the initramfs for?</summary>
The kernel needs drivers to reach the root filesystem (storage controller, LVM, encryption, filesystem), and those drivers are stored on the root filesystem. The initramfs is a small root filesystem loaded into RAM by the boot loader that contains just those pieces, mounts the real root, and hands over.
</details>

<details><summary>Which file do you edit to change GRUB settings, and which do you never edit?</summary>
Edit <code>/etc/default/grub</code> (and use <code>grubby</code> for kernel arguments). Never hand-edit <code>/boot/grub2/grub.cfg</code>: it is generated.
</details>

<details><summary>The machine stops at "Give root password for maintenance". What is the most likely cause and the fix?</summary>
An <code>/etc/fstab</code> entry that cannot be mounted. Log in at the prompt, <code>mount -o remount,rw /</code>, correct or comment out the line (or add <code>nofail</code>), <code>systemctl daemon-reload</code>, then continue or reboot.
</details>

<details><summary>What can <code>journalctl -k -b -1</code> show you that <code>dmesg</code> cannot?</summary>
Kernel messages from the previous boot. <code>dmesg</code> reads the in-memory ring buffer, which starts empty at each boot.
</details>

<details><summary><code>systemd-analyze blame</code> shows a unit took 30 seconds. Did it delay boot by 30 seconds?</summary>
Not necessarily. Units start in parallel. It delayed boot only if it is on the critical chain, meaning something that the default target needs was waiting for it. <code>systemd-analyze critical-chain</code> answers that.
</details>

<details><summary>Why does a logrotate stanza often contain a <code>postrotate</code> script?</summary>
After the log file is renamed, the daemon still holds the old file open and keeps writing to it. The script signals or reloads the daemon so that it reopens the log at the original path.
</details>

<details><summary>You must change a kernel parameter on a remote server. What do you put in place first?</summary>
Out-of-band console access (BMC, serial console, KVM, or someone on site), a note of the current working configuration, and an untouched older kernel entry to fall back to.
</details>

## Cleanup

```bash
sudo rm -rf /var/log/lab12 /etc/logrotate.d/lab12 /etc/logrotate.d/myapp-test
```

Consider keeping `baseline-errors.txt` and `boot-report.sh` somewhere permanent before removing the lab directory. They are useful after every kernel update.

```bash
rm -rf ~/labwork/12
```
