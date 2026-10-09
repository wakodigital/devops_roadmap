# Lab 03 — Shell and Other Basics

**Roadmap topics:** Command Path · Environment Variables · Command Help · Redirects · Super User
**Time:** about 2 hours
**Touches system state:** adds lines to your `~/.bashrc`, and creates then removes one file in `/etc/sudoers.d`.

## What you will be able to do

- Explain exactly how the shell turns the word `ls` into a running program.
- Set, export and persist environment variables, and know which startup file to put them in.
- Find the answer to "how do I use this command" without leaving the terminal.
- Send output, errors and input wherever you want them.
- Use `sudo` deliberately and read its configuration.

## Setup

```bash
mkdir -p ~/labwork/03 && cd ~/labwork/03
```

---

## 1. Command path

When you type a word and press Enter, bash looks for it in this order:

1. **alias** (text substitution defined with `alias`)
2. **function** (defined in the shell)
3. **builtin** (part of bash itself: `cd`, `echo`, `export`, `type`)
4. **hashed path** (a location bash has already looked up and remembered)
5. **each directory in `$PATH`**, left to right, first match wins

If nothing matches: `command not found`.

```bash
echo $PATH
echo $PATH | tr ':' '\n'      # one directory per line

type ls          # probably: ls is aliased to `ls --color=auto'
type -a ls       # every match, in the order the shell would use them
type cd          # cd is a shell builtin
type type
which ls         # searches PATH only; knows nothing about builtins or functions
command -v ls    # the portable way to ask "does this command exist" in scripts
hash             # the shell's cache of command locations
```

`cd` has to be a builtin. A separate program could only change its own working directory, then exit, leaving your shell where it was.

### Running something that is not on PATH

The current directory is deliberately **not** on PATH. Otherwise anyone who can drop a file named `ls` in `/tmp` could have you run it.

```bash
cat > hello <<'EOF'
#!/usr/bin/env bash
echo "hello from $0"
EOF
chmod +x hello

hello          # command not found
./hello        # works: you gave an explicit path
```

### Adding a directory to PATH

```bash
mkdir -p ~/bin
mv hello ~/bin/
export PATH="$HOME/bin:$PATH"     # this shell only
hello                             # now found
```

Order matters. Putting your directory **first** means your versions shadow system commands. Putting it **last** (`PATH="$PATH:$HOME/bin"`) means system commands win. Rocky's default `~/.bashrc` already adds `~/.local/bin` and `~/bin` if they exist, so log out and back in and it will be there without your `export`.

For scripts every user should run, use `/usr/local/bin`. It is already on everyone's PATH.

### Aliases

```bash
alias                      # list current aliases
alias ll='ls -lah'
ll
\ls                        # a leading backslash bypasses the alias once
unalias ll
```

### Exercises 1

1. How many directories are on your PATH? Which one contains `systemctl`?
2. Is `echo` a builtin, a program on disk, or both?
3. Create a script called `ls` in `~/bin` that prints `gotcha`. With `~/bin` first on your PATH, what does `ls` run? Why? Remove the script afterwards.
4. Run `sudo env | grep ^PATH` and compare it with your own PATH. What differs?

<details><summary>Solutions</summary>

```bash
# 1
echo $PATH | tr ':' '\n' | wc -l
command -v systemctl          # /usr/bin/systemctl

# 2
type -a echo
# echo is a shell builtin
# echo is /usr/bin/echo
# Both. The builtin wins because builtins are checked before PATH.

# 3
printf '#!/bin/bash\necho gotcha\n' > ~/bin/ls && chmod +x ~/bin/ls
ls              # probably still the real listing. bash cached /usr/bin/ls the first time
                # you ran it and does not search PATH again.
hash -r         # clear the cache
ls              # gotcha. The alias ls='ls --color=auto' expands first, then the word ls
                # is looked up on PATH, and ~/bin comes before /usr/bin.
type -a ls      # shows the alias, then ~/bin/ls, then /usr/bin/ls
rm ~/bin/ls
hash -r

# 4
# sudo replaces PATH with the secure_path value from /etc/sudoers:
sudo grep secure_path /etc/sudoers
# This is why a command in ~/bin works for you but "sudo mycommand" says command not found.
```

</details>

---

## 2. Environment variables

A **shell variable** exists only in the current shell. An **environment variable** is a shell variable that has been marked for export, so every program the shell starts gets a copy.

```bash
COLOR=blue                 # shell variable. No spaces around =
echo $COLOR
bash -c 'echo "child sees: [$COLOR]"'      # empty: the child did not inherit it

export COLOR               # mark it for export
bash -c 'echo "child sees: [$COLOR]"'      # blue

export STAGE=dev           # define and export in one step
unset COLOR                # remove it
```

Children get a **copy**. A child process can never change its parent's environment. This is why a script cannot `cd` or set variables for the shell that launched it, unless you `source` it.

```bash
env                        # every environment variable
printenv HOME              # one of them
env | sort | less
set | less                 # shell variables too, plus functions (long)
```

Set a variable for a single command, without changing your shell:

```bash
LANG=C ls -l /nonexistent              # error in plain English regardless of locale
TZ=UTC date
TZ=America/Denver date
```

This is the form you see constantly in Docker and CI: `NODE_ENV=production npm start`.

### Variables you should recognise

| Variable | Meaning |
|---|---|
| `HOME` | your home directory |
| `USER` / `LOGNAME` | your username |
| `SHELL` | your login shell |
| `PATH` | where commands are searched for |
| `PWD` / `OLDPWD` | current and previous directory (`cd -` uses `OLDPWD`) |
| `LANG` / `LC_*` | locale: language, sort order, number formats |
| `EDITOR` / `VISUAL` | editor that `sudoedit`, `git commit`, `crontab -e` launch |
| `PS1` | your prompt |
| `TERM` | terminal type |
| `HISTSIZE` / `HISTFILE` | history length and location |
| `?` | exit status of the last command (0 = success) |
| `$` | PID of the current shell |

```bash
echo "$USER on $HOSTNAME using $SHELL, pid $$"
ls /nonexistent; echo "exit status: $?"
ls /etc >/dev/null; echo "exit status: $?"
```

### Making variables permanent: which file?

| File | Read when | Use for |
|---|---|---|
| `/etc/profile`, `/etc/profile.d/*.sh` | every login shell, all users | system-wide environment |
| `~/.bash_profile` | **login** shells (SSH, console) | things to do once per login; on Rocky it sources `~/.bashrc` |
| `~/.bashrc` | **interactive non-login** shells (a new terminal tab, `bash` typed at a prompt) | aliases, functions, prompt, most of your settings |
| `/etc/environment` | read by PAM at login, not by the shell | simple `KEY=value` lines for all users, no shell syntax |

Because Rocky's `~/.bash_profile` sources `~/.bashrc`, putting your settings in `~/.bashrc` covers both cases.

```bash
cat ~/.bash_profile
cat ~/.bashrc

cat >> ~/.bashrc <<'EOF'

# --- linux labs ---
export EDITOR=vim
alias ll='ls -lah'
EOF

source ~/.bashrc           # re-read it in the current shell ("." is the same command)
echo $EDITOR
```

None of these files are read by systemd services or cron jobs. A service gets its environment from its unit file (`Environment=`, `EnvironmentFile=`; Lab 09). This is the usual cause of "it works when I run it by hand but not as a service".

### Exercises 2

1. Create a variable `DEPLOY_ENV` with the value `staging`. Show that a child `bash -c` does not see it, export it, and show that the child now does.
2. Start a child shell with `bash`, change `DEPLOY_ENV` to `prod` inside it, `exit`, and print the value in the parent. Explain the result.
3. Print today's date in UTC and in Tokyo time without changing your shell's `TZ`.
4. Write a script `setvars.sh` containing `export FROM_SCRIPT=yes`. Run it with `bash setvars.sh`, then check `$FROM_SCRIPT`. Then run it with `source setvars.sh` and check again. Explain.

<details><summary>Solutions</summary>

```bash
# 1
DEPLOY_ENV=staging
bash -c 'echo "[$DEPLOY_ENV]"'     # []
export DEPLOY_ENV
bash -c 'echo "[$DEPLOY_ENV]"'     # [staging]

# 2
bash
DEPLOY_ENV=prod
exit
echo $DEPLOY_ENV                   # staging. The child changed its own copy.

# 3
TZ=UTC date; TZ=Asia/Tokyo date

# 4
echo 'export FROM_SCRIPT=yes' > setvars.sh
bash setvars.sh;   echo "[$FROM_SCRIPT]"    # []    ran in a child process
source setvars.sh; echo "[$FROM_SCRIPT]"    # [yes] ran in the current shell
```

</details>

---

## 3. Command help

You will not memorise options. You will memorise how to find them quickly.

```bash
ls --help                 # short usage, printed by the program itself
ls --help | less
man ls                    # the manual page
help cd                   # for bash builtins (they have no man page of their own)
type cd                   # tells you it is a builtin, so you know to use help
```

### Reading a man page

Inside `man` (which uses `less`):

| Key | Action |
|---|---|
| `Space` / `b` | page down / up |
| `/pattern` | search forward; `n` next, `N` previous |
| `g` / `G` | top / bottom |
| `q` | quit |

Go straight to an option: `man ls`, then `/^\s+-t` and Enter finds the line that documents `-t`.

Synopsis notation:

```
tar [OPTION...] [FILE]...
```

`[ ]` is optional, `...` means repeatable, `a|b` means one or the other, and UPPERCASE or underlined words are placeholders you replace.

### Manual sections

The same name can exist in several sections.

| Section | Contents | Example |
|---|---|---|
| 1 | user commands | `man 1 passwd` (the command) |
| 5 | file formats | `man 5 passwd` (the format of `/etc/passwd`) |
| 8 | admin commands | `man 8 useradd` |
| 2, 3 | system calls, C library | `man 2 fork` |
| 7 | overviews and conventions | `man 7 signal`, `man 7 hier` |

```bash
man 5 crontab            # the file format, which is what you usually want
man 1 crontab            # the command
```

### Finding a command when you do not know its name

```bash
sudo mandb               # build the index once (it may already exist)
man -k partition         # search names and one-line descriptions; same as: apropos partition
man -k '^user'           # regex is allowed
whatis tar               # the one-line description
man -f passwd            # every section that has a page called passwd
```

### Other places to look

```bash
ls /usr/share/doc | head                 # packages often ship READMEs and example configs
rpm -qd openssh-server                   # documentation files belonging to a package
info coreutils                           # GNU's longer-form manuals (arrow keys, Enter, q)
```

`tldr` (community-maintained examples for common commands) is the fastest route for "just show me the usual invocation". It is in EPEL; whether to add EPEL is a decision for Lab 10.

### Exercises 3

Answer using only `man`, `--help` and `help`.

1. Which `ls` option sorts by file extension?
2. Which `df` option shows inode usage rather than block usage?
3. Which manual section documents the format of `/etc/fstab`? What is the sixth field for?
4. Find a command that reports "block device attributes" using `man -k`.
5. What does `cd -P` do?

<details><summary>Solutions</summary>

1. `-X`
2. `-i`
3. Section 5 (`man 5 fstab`). The sixth field is `fs_passno`: the order in which `fsck` checks filesystems at boot (0 = do not check).
4. `man -k "block device"` → `blkid` ("locate/print block device attributes") and `lsblk`.
5. `help cd` → use the physical directory structure, resolving symbolic links, instead of following them logically.

</details>

---

## 4. Redirects

Every process starts with three open streams, each with a number called a file descriptor.

| FD | Name | Default | Purpose |
|---|---|---|---|
| 0 | stdin | keyboard | input |
| 1 | stdout | terminal | normal output |
| 2 | stderr | terminal | errors and diagnostics |

Separating stdout from stderr is what lets you pipe a command's results onward while still seeing its errors on screen.

| Syntax | Effect |
|---|---|
| `cmd > file` | stdout to file, **overwriting** |
| `cmd >> file` | stdout to file, appending |
| `cmd 2> file` | stderr to file |
| `cmd > out 2> err` | each to its own file |
| `cmd > file 2>&1` | both to file ("send 2 to wherever 1 is going now") |
| `cmd &> file` | bash shorthand for the line above |
| `cmd &>> file` | both, appending |
| `cmd 2>/dev/null` | discard errors |
| `cmd < file` | file to stdin |
| `cmd <<EOF … EOF` | here-document: the following lines become stdin |
| `cmd <<< "text"` | here-string: the text becomes stdin |
| `cmd1 \| cmd2` | stdout of cmd1 to stdin of cmd2 (Lab 05) |
| `cmd1 \|& cmd2` | stdout **and** stderr of cmd1 to cmd2 |

```bash
cd ~/labwork/03
ls /etc/hostname /nonexistent                     # one line of each kind on screen
ls /etc/hostname /nonexistent > out.txt           # error still on screen
ls /etc/hostname /nonexistent 2> err.txt          # result still on screen
ls /etc/hostname /nonexistent > out.txt 2> err.txt
cat out.txt; cat err.txt
ls /etc/hostname /nonexistent &> both.txt
cat both.txt
```

### Order matters

Redirections are processed left to right.

```bash
ls /etc/hostname /nonexistent > a.txt 2>&1     # 1 → a.txt, then 2 → where 1 is (a.txt). Both in file.
ls /etc/hostname /nonexistent 2>&1 > b.txt     # 2 → where 1 is (terminal), then 1 → b.txt. Error on screen.
```

### Here-documents

You have already used these to create lab files.

```bash
cat > app.conf <<EOF
user=$USER
home=$HOME
EOF
cat app.conf                 # variables were expanded

cat > app.conf <<'EOF'
user=$USER
home=$HOME
EOF
cat app.conf                 # quoted delimiter: text is taken literally
```

Quote the delimiter whenever the content contains `$` or backticks that should be written as-is, which is the usual case when a heredoc writes a script.

### /dev/null and friends

```bash
grep -r "PermitRoot" /etc 2>/dev/null          # hide the "Permission denied" noise
command -v docker >/dev/null && echo "docker is installed"
> empty.txt                                    # truncate (or create) a file with no command at all
```

### The sudo redirect trap

```bash
sudo echo "test" > /etc/labtest        # Permission denied
```

The redirect is performed by **your** shell, before `sudo` runs, and your shell is not root. Two fixes:

```bash
echo "test" | sudo tee /etc/labtest > /dev/null      # tee runs as root and does the writing
echo "more" | sudo tee -a /etc/labtest > /dev/null   # -a appends
sudo rm /etc/labtest
# or: sudo sh -c 'echo "test" > /etc/labtest'
```

### Protecting yourself from `>`

```bash
set -o noclobber
echo one > keep.txt
echo two > keep.txt          # cannot overwrite existing file
echo two >| keep.txt         # >| overrides the protection deliberately
set +o noclobber
```

### Exercises 4

1. Run `find /etc -name "*.conf"` as your normal user so that the file list goes to `conf-list.txt` and the permission errors go to `conf-errors.txt`. How many of each?
2. Do the same search, discard the errors, and only print the count of files found.
3. Append the current date to `conf-list.txt` without disturbing its contents.
4. Write a heredoc that creates `motd.txt` containing two lines: `Host: <your hostname>` and `Cost: $0` (a literal dollar sign and zero, with the hostname expanded).
5. Explain what this does and why it is a common idiom in cron jobs: `/usr/local/bin/backup.sh >> /var/log/backup.log 2>&1`

<details><summary>Solutions</summary>

```bash
# 1
find /etc -name "*.conf" > conf-list.txt 2> conf-errors.txt
wc -l conf-list.txt conf-errors.txt

# 2
find /etc -name "*.conf" 2>/dev/null | wc -l

# 3
date >> conf-list.txt

# 4
cat > motd.txt <<EOF
Host: $HOSTNAME
Cost: \$0
EOF
# unquoted delimiter so $HOSTNAME expands; the backslash keeps $0 literal

# 5
# Appends both normal output and errors to one log file. Cron has no terminal,
# so anything not redirected is either mailed to the user or lost.
```

</details>

---

## 5. Super user

`root` (UID 0) bypasses all permission checks. The working practice is to log in as yourself and borrow root's privileges one command at a time, so that there is a named person and a log entry behind every privileged action.

| Command | What you get |
|---|---|
| `sudo cmd` | run one command as root, authenticated with **your** password |
| `sudo -u bob cmd` | run one command as another user |
| `sudo -i` | an interactive root login shell (root's environment and home) |
| `sudo -s` | a root shell that keeps much of your environment |
| `su -` | become root using **root's** password (often disabled or unknown) |
| `su - bob` | become bob using bob's password; `sudo su - bob` uses yours |
| `sudo -l` | list what you are allowed to run |
| `sudo -k` | forget the cached credential; the next sudo asks again |
| `sudo -v` | refresh the cached credential without running anything |
| `sudo !!` | re-run the previous command with sudo |
| `sudoedit file` | edit a file as root safely (Lab 02) |

```bash
whoami
sudo whoami
sudo -l
id                             # are you in the wheel group?
sudo -u nobody whoami
```

### How it is configured

On Rocky, membership of the **`wheel`** group grants full sudo. (Debian and Ubuntu use a group called `sudo`.)

```bash
sudo grep -v '^#' /etc/sudoers | grep -v '^$'
sudo ls -l /etc/sudoers.d/
getent group wheel
```

A sudoers rule has this shape:

```
who    where=(as_whom)    what
%wheel ALL=(ALL)          ALL
walter ALL=(ALL)          NOPASSWD: /usr/bin/systemctl restart nginx
```

`%` marks a group. `NOPASSWD:` skips the password prompt for the commands after it. Automation accounts (such as the one Ansible connects as) commonly have `NOPASSWD: ALL`.

Rules for changing it:

- Only ever through `visudo`, which refuses to save a file with a syntax error.
- Put your own rules in a file under `/etc/sudoers.d/` and leave `/etc/sudoers` untouched. Package updates will not overwrite your file, and removing it undoes your change.
- Filenames in `sudoers.d` must not contain a `.` or end in `~`, or they are silently ignored.
- Use full paths for commands.

### A scoped rule, start to finish

Give yourself password-free access to exactly one harmless command, prove it works, and remove it.

```bash
sudo visudo -f /etc/sudoers.d/90-lab
```

Add this one line (replace `walter` with the output of `whoami`), save and quit:

```
walter ALL=(root) NOPASSWD: /usr/bin/systemctl status sshd
```

```bash
sudo visudo -c                        # check every sudoers file: "parsed OK"
sudo -k                               # drop the cached password
sudo systemctl status sshd            # no password prompt
sudo systemctl status crond           # prompts: not covered by the rule
sudo -l                               # the new rule is listed
sudo rm /etc/sudoers.d/90-lab         # clean up
```

If neither command prompts for a password, your account already has a `NOPASSWD` rule (common on machines set up for Ansible). `sudo -l` will show it.

### Where sudo is logged

Every sudo invocation is recorded, including failures.

```bash
sudo grep sudo /var/log/secure | tail -5
sudo journalctl _COMM=sudo --since today | tail -10
```

Each entry shows who ran it, from which terminal, in which directory, as which user, and the full command. Lab 06 uses this log for auditing.

### Habits

- Do not work in a root shell for convenience. `sudo -i` is for a sequence of root-only steps, and you `exit` when finished.
- Read a command before putting `sudo` in front of it, especially one pasted from the web. `curl … | sudo bash` runs a stranger's script as root.
- Keep `PermitRootLogin no` in `sshd_config`, so that every root action traces back through a named account.
- `sudo` changes PATH (`secure_path`) and strips most of your environment. If a variable has to pass through, use `sudo VAR=value cmd` or `sudo -E`.

### Exercises 5

1. Which groups are you in, and which one grants you sudo?
2. Run `cat /etc/shadow` as yourself, then repeat it using `sudo !!`. What is this file and why is it protected?
3. Create a file `/root/labfile` containing the text `made by sudo`, using `tee`. Read it back, then delete it.
4. Run `sudo -i`, then compare `pwd`, `whoami` and `echo $PATH` with your normal shell. `exit` when done.
5. Find your last five sudo commands in the logs.

<details><summary>Solutions</summary>

```bash
# 1
id -nG                   # look for wheel

# 2
cat /etc/shadow          # Permission denied
sudo !!                  # holds password hashes; readable only by root so they cannot be cracked offline

# 3
echo "made by sudo" | sudo tee /root/labfile > /dev/null
sudo cat /root/labfile
sudo rm /root/labfile

# 4
sudo -i                  # pwd → /root, whoami → root, PATH includes /root/bin and sbin directories
exit

# 5
sudo journalctl _COMM=sudo -n 20 --no-pager | grep COMMAND | tail -5
```

</details>

---

## Check yourself

<details><summary>Why does <code>./script.sh</code> work when <code>script.sh</code> says "command not found"?</summary>
A name with no slash is looked up only in aliases, functions, builtins and PATH. The current directory is not on PATH. <code>./</code> makes it an explicit path, so no lookup happens.
</details>

<details><summary>You set <code>API_KEY=abc</code> in your shell, then run a Python script that reports the variable is missing. Why?</summary>
The variable was never exported, so it is a shell variable and child processes do not receive it. Use <code>export API_KEY=abc</code>, or <code>API_KEY=abc python script.py</code>.
</details>

<details><summary>What is the difference between <code>cmd > f 2>&1</code> and <code>cmd 2>&1 > f</code>?</summary>
In the first, stdout goes to the file and then stderr is pointed at the same place: both end up in the file. In the second, stderr is pointed at the terminal (where stdout currently goes) and then stdout is moved to the file: errors stay on screen.
</details>

<details><summary>Why does <code>sudo echo x > /etc/file</code> fail?</summary>
The redirection is set up by your unprivileged shell before sudo starts. Only <code>echo</code> runs as root. Pipe into <code>sudo tee</code> so that the process doing the writing is the privileged one.
</details>

<details><summary>Your service runs fine from your shell but fails under systemd with a missing environment variable. Where do you look?</summary>
systemd does not read <code>~/.bashrc</code> or <code>/etc/profile</code>. The variable has to be set in the unit with <code>Environment=</code> or <code>EnvironmentFile=</code>.
</details>

## Cleanup

```bash
rm -r ~/labwork/03
rm -f ~/bin/hello
ls /etc/sudoers.d/           # confirm 90-lab is gone
```

Keep the `EDITOR` and `ll` lines in `~/.bashrc`.
