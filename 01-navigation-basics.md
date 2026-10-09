# Lab 01 — Navigation Basics

**Roadmap topics:** Basic Commands · Moving Files / Directories · Creating & Deleting Files / Dirs · Directory Hierarchy Overview
**Time:** about 90 minutes
**Touches system state:** no. Everything happens in `~/labwork/01`.

## What you will be able to do

- Move around the filesystem without thinking about it, using absolute and relative paths.
- Read every column of `ls -l`.
- Create, move, rename and delete files and directories, including the awkward cases (spaces, leading dashes, non-empty directories).
- Say what belongs in `/etc`, `/var`, `/usr`, `/opt`, `/proc`, `/run` and why.

## Setup

```bash
mkdir -p ~/labwork/01 && cd ~/labwork/01
sudo dnf install -y tree        # small, harmless, used below
```

---

## 1. Basic commands

### Where am I, who am I

```bash
pwd            # print working directory
whoami         # current user
id             # uid, gid, and every group you belong to
hostname       # this machine's name
date           # current date and time
uname -r       # running kernel version
cat /etc/os-release
```

### Paths

A path is either **absolute** (starts at `/`, means the same thing from anywhere) or **relative** (starts from where you are now).

| Symbol | Meaning |
|---|---|
| `/` | the root of the filesystem |
| `.` | the current directory |
| `..` | the parent directory |
| `~` | your home directory (`/home/you`) |
| `~bob` | bob's home directory |
| `-` | (only with `cd`) the directory you were in before |

```bash
cd /var/log        # absolute
cd ..              # now in /var
cd log             # relative: back in /var/log
cd                 # no argument = go home
cd -               # back to /var/log, and it prints where it took you
cd ~/labwork/01
```

### Listing

```bash
ls                 # names only
ls -l              # long format
ls -a              # include dotfiles (hidden files start with .)
ls -lh             # human-readable sizes
ls -lt             # sort by modification time, newest first
ls -ltr            # same, reversed: newest at the bottom (handy in big dirs)
ls -lS             # sort by size
ls -d /etc/ss*     # show the directory entries themselves, not their contents
ls -R              # recurse
ls -i              # show inode numbers (you will need these in Lab 04 and 11)
```

Reading one line of `ls -l`:

```
-rw-r--r--. 1 walter walter 1482 Oct  8 20:15 notes.txt
│└───┬───┘│ │ └──┬─┘ └──┬─┘ └┬─┘ └────┬─────┘ └───┬───┘
│    │    │ │    │      │    │        │           └ name
│    │    │ │    │      │    │        └ last modified
│    │    │ │    │      │    └ size in bytes
│    │    │ │    │      └ owning group
│    │    │ │    └ owning user
│    │    │ └ link count (hard links; for a directory, 2 + number of subdirectories)
│    │    └ a trailing "." means an SELinux context is set; "+" means ACLs are set
│    └ permissions: user / group / other (Lab 04)
└ type: - file, d directory, l symlink, c char device, b block device, s socket, p pipe
```

The trailing `.` is something you will see on Rocky and not on Ubuntu, because SELinux is enforcing by default on Rocky.

### Looking at files

```bash
cat /etc/hostname          # dump a small file
less /etc/services         # page a big one: space/b = page down/up, /text = search, n = next, q = quit
head -n 5 /etc/passwd      # first 5 lines
tail -n 5 /etc/passwd      # last 5 lines
file /usr/bin/ls /etc/passwd /etc    # what kind of thing is this?
stat /etc/passwd           # everything the filesystem knows about the file
wc -l /etc/passwd          # count lines
```

### Keys worth building into muscle memory

| Key | Effect |
|---|---|
| `Tab` | complete a command or path; press twice to list candidates |
| `Ctrl-R` | search backwards through history; press again for the next match |
| `Ctrl-A` / `Ctrl-E` | jump to start / end of line |
| `Ctrl-W` | delete the word before the cursor |
| `Ctrl-U` | delete from cursor to start of line |
| `Ctrl-L` | clear the screen |
| `Ctrl-C` | cancel the current command |
| `Ctrl-D` | end of input; on an empty prompt, log out |
| `!!` | the previous command (most often as `sudo !!`) |
| `!$` | the last argument of the previous command |

```bash
history | tail -20
history | grep dnf
```

### Wildcards (globbing)

The **shell** expands these before the command ever runs. `ls` never sees the `*`.

| Pattern | Matches |
|---|---|
| `*` | any run of characters (not a leading dot) |
| `?` | exactly one character |
| `[abc]` / `[a-z]` | one character from the set / range |
| `[!abc]` | one character not in the set |
| `{a,b,c}` | brace expansion: generates each alternative (not a match, a text generator) |
| `{1..5}` | generates 1 2 3 4 5 |

```bash
ls /etc/*.conf
ls /etc/host*
ls -d /etc/[a-c]*
echo file{1..5}.txt          # see what the shell generates
echo /etc/{hosts,hostname}
```

`echo` in front of a command is the safest way to preview what a glob will expand to before you hand it to `rm`.

### Exercises 1

1. From your home directory, go to `/usr/share/doc` using an absolute path, then to `/usr/share/man` using a relative path, then return home with one two-character command.
2. List `/var/log` so the most recently modified entry is the last line printed, with human-readable sizes.
3. How many entries in `/etc` end in `.conf`? (Count only the ones directly in `/etc`.)
4. What type of file is `/dev/null`? What about `/dev/sda` or `/dev/nvme0n1` (whichever exists)? Use `ls -l` and read the first character.
5. Without running it, predict what `echo {a,b}{1,2}` prints. Then run it.

<details><summary>Solutions</summary>

```bash
# 1
cd /usr/share/doc
cd ../man
cd            # or: cd ~

# 2
ls -ltrh /var/log

# 3
ls -d /etc/*.conf | wc -l

# 4
ls -l /dev/null        # starts with c: character device
lsblk                  # find your disk name first
ls -l /dev/nvme0n1     # starts with b: block device

# 5
a1 a2 b1 b2
```

</details>

---

## 2. Creating and deleting files and directories

```bash
cd ~/labwork/01

touch a.txt                     # create an empty file (or update its timestamp if it exists)
touch b.txt c.txt
echo "hello" > d.txt            # create a file with content (> overwrites)
echo "again" >> d.txt           # >> appends

mkdir projects
mkdir projects/web/static       # FAILS: parent projects/web does not exist
mkdir -p projects/web/static    # -p creates parents as needed, and is silent if it already exists
mkdir -p projects/{api,db}/{src,tests}
tree projects
```

Deleting:

```bash
rm a.txt                # delete a file. There is no trash can and no undo.
rm -i b.txt             # ask first
rmdir projects/db/tests # removes a directory only if it is empty
rmdir projects/api      # FAILS: not empty
rm -r projects/api      # recursive: the directory and everything under it
rm -rf projects/db      # -f: never prompt, ignore missing files
```

**How `rm -rf` goes wrong, and the habits that prevent it**

- An unset variable: `rm -rf "$DIR"/*` with `DIR` empty becomes `rm -rf /*`. In scripts use `set -u`, or `"${DIR:?}"` which aborts if empty (Lab 14).
- A stray space: `rm -rf ~/labwork /01` deletes all of `~/labwork` and then complains that `/01` does not exist.
- Preview first: run `ls` or `echo` with the same arguments, then press up-arrow and change the command to `rm`.
- Use Tab completion for paths, so that you only type paths that exist.

### Awkward names

```bash
touch "my file.txt"       # a name with a space
ls -l
rm my file.txt            # FAILS: looks for "my" and "file.txt"
rm "my file.txt"          # quote it
touch my\ file.txt        # or escape the space
rm my\ file.txt

touch -- -oops            # a name starting with a dash
rm -oops                  # FAILS: rm thinks -oops is a set of options
rm -- -oops               # "--" means "no more options after this"
# alternative: rm ./-oops
```

`--` works with almost every command and is worth remembering for scripts that handle arbitrary filenames.

### Exercises 2

1. With **one** command, create this tree: `site/css`, `site/js`, `site/img/icons`.
2. Create ten empty files `log1.txt` … `log10.txt` in `site/` with one command.
3. Delete `log1.txt`, `log2.txt` and `log3.txt` (and only those) using a pattern. Preview with `echo` first.
4. `log1*` would have been a mistake in exercise 3. Why?
5. Delete the whole `site` directory.

<details><summary>Solutions</summary>

```bash
# 1
mkdir -p site/{css,js,img/icons}

# 2
touch site/log{1..10}.txt

# 3
echo site/log[1-3].txt
rm site/log[1-3].txt

# 4
# log1* also matches log10.txt

# 5
rm -r site
```

</details>

---

## 3. Moving files and directories

`mv` does two jobs with one mechanism: it changes a file's path. If the new path is in the same directory, that is a rename. If it is in a different directory, that is a move.

```bash
cd ~/labwork/01
mkdir -p inbox archive
touch inbox/report-{jan,feb,mar}.txt

mv inbox/report-jan.txt archive/               # move into a directory (trailing / makes intent explicit)
mv inbox/report-feb.txt inbox/february.txt     # rename in place
mv inbox/report-mar.txt archive/march.txt      # move and rename at once
mv inbox/*.txt archive/                        # move many: the last argument must be a directory
mv archive old-archive                         # rename a directory
tree
```

Things that bite:

```bash
touch x.txt y.txt
mv x.txt y.txt        # y.txt is silently replaced. The old y.txt is gone.
touch x.txt
mv -i x.txt y.txt     # -i asks before overwriting. Answer n.
mv -n x.txt y.txt     # -n never overwrites
mv -v x.txt z.txt     # -v reports what it did
```

Within one filesystem, `mv` is instant no matter how big the file is, because only the directory entry changes. Across filesystems (for example `/home` to `/tmp` when they are separate mounts) it has to copy the data and then delete the original.

Copying (`cp`) is covered in Lab 04 with links and permissions, because the interesting parts of `cp` are about what metadata it preserves.

### Exercises 3

1. Create `stage/` containing `app.conf`, `app.log`, `db.conf`, `db.log`. Move all `.log` files into a new `logs/` directory and all `.conf` files into a new `conf/` directory.
2. Rename `conf/app.conf` to `conf/application.conf`.
3. Move `conf` inside `stage` so that the result is `stage/conf/…`.
4. What does `mv logs stage` do if `stage/logs` already exists as an empty directory? What if it exists and is not empty? Try both.

<details><summary>Solutions</summary>

```bash
# 1
mkdir stage logs conf
touch stage/{app,db}.{conf,log}
mv stage/*.log logs/
mv stage/*.conf conf/

# 2
mv conf/app.conf conf/application.conf

# 3
mv conf stage/

# 4
# Empty target: logs replaces stage/logs.
# Non-empty target: "mv: cannot move 'logs' to 'stage/logs': Directory not empty"
# mv will not merge directories. Use rsync -a or cp -a for that (Lab 04).
```

</details>

---

## 4. Directory hierarchy overview

Linux has one tree rooted at `/`. Disks, USB sticks and network shares are attached ("mounted") at directories inside that tree; there are no drive letters. The layout is a convention called the Filesystem Hierarchy Standard.

| Path | What lives there | On your tower |
|---|---|---|
| `/etc` | System-wide configuration, plain text | `sshd_config`, `fstab`, systemd unit overrides, dnf repos |
| `/var` | Data that changes while the system runs | `/var/log`, `/var/lib/docker`, package caches |
| `/var/log` | Logs | `secure`, `messages`, `dnf.log` |
| `/home` | Users' home directories | yours |
| `/root` | root's home directory (not under `/home`, so it is available if `/home` fails to mount) | |
| `/usr` | Installed software, read-only in normal operation | `/usr/bin`, `/usr/lib`, `/usr/share` |
| `/usr/local` | Software you installed by hand, outside the package manager | scripts you write go in `/usr/local/bin` |
| `/opt` | Self-contained third-party software | |
| `/bin`, `/sbin`, `/lib`, `/lib64` | On Rocky these are symlinks into `/usr` | run `ls -ld /bin` |
| `/boot` | Kernel, initramfs, bootloader files | Lab 12 |
| `/dev` | Device files: disks, terminals, `/dev/null` | |
| `/proc` | A virtual filesystem: the kernel's view of processes and itself | `/proc/cpuinfo`, `/proc/<pid>/` |
| `/sys` | A virtual filesystem: devices, drivers, cgroups | `/sys/fs/cgroup` (Lab 16) |
| `/run` | Runtime state since boot (PIDs, sockets), held in RAM | `/run/docker.sock` |
| `/tmp` | Temporary files, world-writable, cleaned automatically | |
| `/var/tmp` | Temporary files that should survive a reboot | |
| `/srv` | Data served by this machine (by convention; often unused) | |
| `/mnt`, `/media` | Mount points for temporary mounts / removable media | you will mount lab disks under `/mnt` in Lab 11 |

A working rule for a sysadmin: **configuration is in `/etc`, state is in `/var`, programs are in `/usr`.** If you back up `/etc`, `/var` (selectively) and `/home`, you can rebuild the machine. That is the same split Ansible relies on when it templates files into `/etc`.

```bash
ls -ld /bin /sbin /lib /lib64     # symlinks: the "merged /usr" layout
man hier                          # the manual page that describes the hierarchy
df -h                             # which directories are separate filesystems
findmnt -t xfs,ext4               # real on-disk filesystems only
ls /proc | head -20               # numbered directories are processes
grep -m1 "model name" /proc/cpuinfo                # exactly which CPU is in this machine
cat /proc/meminfo | head -3
du -sh /var/log /var/lib/docker 2>/dev/null        # du needs sudo to see everything
```

### Exercises 4

1. In which directory would you expect to find each of these? Predict, then verify.
   a. the SSH server's configuration file
   b. the log of package installs
   c. the `ls` program itself
   d. Docker's stored images and volumes
   e. the kernel image the machine booted from
2. `/proc/cpuinfo` shows a size of 0 in `ls -l` but `cat` prints plenty. Why?
3. Is `/tmp` on your tower a real disk directory or held in memory? How do you tell?
4. Find the three largest directories directly under `/var`. (You will need `sudo`.)

<details><summary>Solutions</summary>

```bash
# 1
ls -l /etc/ssh/sshd_config        # a
ls -l /var/log/dnf.log            # b
type ls; ls -l /usr/bin/ls        # c
sudo ls /var/lib/docker           # d
ls -l /boot/vmlinuz-*             # e

# 2
# /proc is not stored on disk. The kernel generates the content at the moment you read it,
# so there is no stored size to report.

# 3
findmnt /tmp
# No output: /tmp is just a directory on the root filesystem (the Rocky default).
# A line showing "tmpfs": it is held in memory.

# 4
sudo du -h --max-depth=1 /var 2>/dev/null | sort -h | tail -4
# the last line is /var itself, so the three above it are the answer
```

</details>

---

## Challenge

Without using a file manager or an editor, and using as few commands as you can:

1. Build this under `~/labwork/01/challenge`:

   ```
   challenge/
   ├── 2026/
   │   ├── q1/  q2/  q3/  q4/
   ├── drafts/
   │   ├── note-01.md … note-12.md
   └── final/
   ```

2. Move notes 01–03 into `2026/q1`, 04–06 into `q2`, 07–09 into `q3`, 10–12 into `q4`.
3. Rename `drafts` to `empty-drafts`, then remove it with the command that only succeeds on an empty directory.
4. Show the result with `tree`.

<details><summary>One solution</summary>

```bash
cd ~/labwork/01
mkdir -p challenge/{2026/q{1..4},drafts,final}
cd challenge
touch drafts/note-{01..12}.md
mv drafts/note-0[1-3].md 2026/q1/
mv drafts/note-0[4-6].md 2026/q2/
mv drafts/note-0[7-9].md 2026/q3/
mv drafts/note-1[0-2].md 2026/q4/
mv drafts empty-drafts
rmdir empty-drafts
tree
```

</details>

## Check yourself

Answer aloud before opening.

<details><summary>What is the difference between <code>rm -r dir</code> and <code>rmdir dir</code>?</summary>
<code>rmdir</code> only removes an empty directory and fails otherwise, which makes it a safe way to assert "there is nothing left in here". <code>rm -r</code> removes the directory and everything beneath it.
</details>

<details><summary>You type <code>ls *.log</code> in a directory with no .log files and get "cannot access '*.log'". Who printed that, and why does it mention the literal asterisk?</summary>
<code>ls</code> printed it. The shell found no match, so by default bash passes the pattern through unchanged, and <code>ls</code> went looking for a file literally named <code>*.log</code>.
</details>

<details><summary>Why is <code>/root</code> not <code>/home/root</code>?</summary>
So that root can log in and repair the system even when <code>/home</code> is a separate filesystem that failed to mount.
</details>

<details><summary>Where would you put a script you wrote that every user on the machine should be able to run?</summary>
<code>/usr/local/bin</code>. It is on everyone's PATH and the package manager never touches it.
</details>

## Cleanup

```bash
rm -r ~/labwork/01
```
