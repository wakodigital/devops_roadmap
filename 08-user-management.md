# Lab 08 — User Management

**Roadmap topics:** Create / Delete / Update · Users and Groups · Managing Permissions
**Time:** about 2.5 hours
**Touches system state:** yes. Creates users and groups whose names all start with `lab`, a directory `/srv/labshare`, and one file in `/etc/sudoers.d`. The Cleanup section removes all of them. Your own account and existing service accounts are not modified.

## What you will be able to do

- Create, modify, lock and remove user accounts and groups, and read the four files that define them.
- Set password ageing and expiry policy.
- Build a shared project directory that works properly for a team.
- Grant access to one extra user with an ACL when owner/group/other is not enough.
- Delegate one specific privileged command to a group through sudo.

## Setup

```bash
mkdir -p ~/labwork/08 && cd ~/labwork/08
sudo dnf install -y acl
getent passwd | grep -c '^lab'    # expect 0: confirm no existing account names start with "lab"
sudo sshd -T | grep -i '^passwordauthentication'
```

This lab gives throwaway accounts simple passwords. That is acceptable only while SSH password logins are off (`passwordauthentication no` above) or the SSH port is unreachable from outside. If yours says `yes`, choose your own strong passwords in place of the ones printed here, and do the Cleanup section the same day.

---

## 1. Where accounts live

A user is, to the kernel, a number: the **UID**. Names exist for humans and are mapped through four text files.

```bash
grep "^$USER:" /etc/passwd
```

```
walter:x:1000:1000:Walter:/home/walter:/bin/bash
   1   2   3    4     5         6          7
```

| Field | Meaning |
|---|---|
| 1 | login name |
| 2 | `x`: the password hash is in `/etc/shadow` |
| 3 | UID |
| 4 | primary GID |
| 5 | comment (GECOS): full name |
| 6 | home directory |
| 7 | login shell. `/sbin/nologin` or `/bin/false` = cannot log in interactively |

```bash
sudo grep "^$USER:" /etc/shadow
```

```
walter:$6$Qd…:20360:0:99999:7:::
   1     2      3   4   5   6 7 8
```

| Field | Meaning |
|---|---|
| 1 | login name |
| 2 | password hash. `$6$` = SHA-512, `$y$` = yescrypt. `!` or `*` at the start = no usable password (locked or never set) |
| 3 | date of last password change, in days since 1970-01-01 |
| 4 | minimum days between changes |
| 5 | maximum days before a change is required |
| 6 | warning days before expiry |
| 7 | days of inactivity after expiry before the account is disabled |
| 8 | account expiry date (days since 1970) |

```bash
grep "^wheel:" /etc/group          # name : x : GID : comma-separated supplementary members
sudo grep "^wheel:" /etc/gshadow   # group passwords and administrators (rarely used)
ls -l /etc/passwd /etc/shadow /etc/group /etc/gshadow
```

`/etc/passwd` is world-readable, because every `ls -l` needs it to turn UIDs into names. `/etc/shadow` is readable only by root, so the hashes cannot be taken away and cracked offline.

Query through `getent` and `id` rather than grepping the files. They also consult any other configured source (LDAP, SSSD), where the files alone would miss users.

```bash
getent passwd $USER
getent group wheel
id                      # your uid, gid and all groups
id root
id -u; id -un; id -Gn
```

### UID ranges and defaults

| Range | Used for |
|---|---|
| 0 | root |
| 1–999 | system accounts: daemons and services, no login |
| 1000 and up | human users |

```bash
grep -E '^(UID|GID|SYS_UID|SYS_GID)_(MIN|MAX)|^PASS_|^UMASK|^CREATE_HOME|^USERGROUPS_ENAB' /etc/login.defs
useradd -D              # defaults for new accounts
ls -la /etc/skel        # files copied into every new home directory
awk -F: '$3 >= 1000 && $3 < 60000 {print $1, $3, $7}' /etc/passwd     # the human accounts
awk -F: '$3 < 1000 && $7 !~ /nologin|false|sync|shutdown|halt/' /etc/passwd   # system accounts with a real shell
```

The second `awk` is a review check. Apart from root, it should print nothing.

---

## 2. Create, update, delete

### Creating

```bash
sudo useradd labalice                       # Rocky creates a home dir and a same-named group by default
getent passwd labalice
sudo ls -la /home/labalice                  # populated from /etc/skel
sudo grep labalice /etc/shadow              # "!!" in the hash field: no password yet, so no password login

sudo passwd labalice                        # set one interactively
sudo passwd -S labalice                     # status: PS = password set, LK = locked, NP = no password
```

Common options, all at once:

```bash
sudo useradd \
  -c "Bob Lab" \
  -s /bin/bash \
  -G games \
  -e 2027-01-01 \
  labbob
#  -c comment   -s shell   -G supplementary groups (never wheel for a lab account)   -e account expiry date
#  -m create home (already the default on Rocky; NOT the default on Debian/Ubuntu)
#  -d /path     home directory elsewhere
#  -u 1500      a specific UID       -g group   primary group
#  -M           no home directory    -r         system account
id labbob
```

`useradd` is the low-level tool and exists everywhere. Debian and Ubuntu also have a friendlier interactive `adduser`; on Rocky, `adduser` is just a symlink to `useradd`.

Setting a password without a prompt, for scripts (never put a real password on a command line; it lands in history and in `ps`):

```bash
echo 'labbob:Temp-Pass-4821' | sudo chpasswd
sudo chage -d 0 labbob                      # force a password change at first login
```

### System (service) accounts

A daemon should run as its own unprivileged user, so that a compromise of the service is not a compromise of the machine.

```bash
sudo useradd -r -s /sbin/nologin -d /var/lib/labsvc -c "Lab service account" labsvc
getent passwd labsvc                        # UID below 1000, nologin shell
sudo -u labsvc whoami                       # root can still run commands as it
sudo su - labsvc                            # "This account is currently not available."
```

`-r` picks a UID from the system range and does not create a home directory. You will run a systemd service as this account in Lab 09.

### Updating

```bash
sudo usermod -c "Alice Lab" labalice        # change the comment
sudo usermod -s /bin/sh labalice            # change the shell
sudo usermod -l labally labalice            # rename the LOGIN (home dir and group keep the old name)
sudo usermod -l labalice labally            # rename back
sudo usermod -d /home/alice-new -m labalice # move the home directory, with contents
sudo usermod -d /home/labalice -m labalice  # and back
cat /etc/shells                             # the shells an account may be given
```

### Locking and expiring

```bash
sudo usermod -L labalice                    # lock: prefixes the hash with "!"
sudo passwd -S labalice                     # LK
sudo usermod -U labalice                    # unlock
```

**Locking the password does not block SSH key logins.** To disable an account fully, expire it:

```bash
sudo usermod -e 1 labalice                  # account expiry = day 1 = long past → all logins refused
sudo chage -l labalice | grep 'Account expires'
sudo usermod -e '' labalice                 # remove the expiry
# also worth doing for a leaver:  sudo usermod -s /sbin/nologin USER   and remove ~USER/.ssh/authorized_keys
```

### Password ageing

```bash
sudo chage -l labalice                      # show the policy for one account
sudo chage -M 90 -m 1 -W 14 labalice        # max 90 days, min 1 day between changes, warn 14 days ahead
sudo chage -E 2027-06-30 labalice           # account expiry date
sudo chage -d 0 labalice                    # must change password at next login
sudo chage -l labalice
```

Defaults for future accounts are the `PASS_*` lines in `/etc/login.defs`. Complexity rules live in `/etc/security/pwquality.conf`.

### Deleting

```bash
sudo useradd labtemp
sudo touch /tmp/labtemp-file && sudo chown labtemp /tmp/labtemp-file
sudo userdel labtemp                        # removes the account, KEEPS /home/labtemp and mail spool
ls -ld /home/labtemp /tmp/labtemp-file      # owner shown as a bare number: the UID has no name any more
```

Those orphaned files are a real hazard: the next account created may be given the same UID and silently inherit them.

```bash
sudo find / -xdev -nouser -o -xdev -nogroup 2>/dev/null | head     # ownerless files: a review check
sudo rm -rf /home/labtemp /tmp/labtemp-file /var/spool/mail/labtemp
sudo userdel -r labbob                      # -r removes the home directory and mail spool too
```

Before deleting a real account: kill its processes (`sudo pkill -u USER`), check for cron jobs (`sudo crontab -u USER -l`), and decide what to do with files it owns outside its home.

### Exercises 2

1. Create `labcarol` with the comment "Carol Lab", shell `/bin/bash`, and an account expiry date 30 days from today. (Hint: `date -d '+30 days' +%F`.)
2. Set a password for her non-interactively and force a change at first login. Confirm with `chage -l`.
3. Test it: `su - labcarol` from your own account. What happens?
4. Lock her password. Show the difference in `/etc/shadow` before and after.
5. Create a system account `labworker` with no login shell and no home directory. What UID did it get, and why is that different from labcarol's?
6. Which accounts on the tower currently have a usable password? (`passwd -S` on each, or read field 2 of shadow.)

<details><summary>Solutions</summary>

```bash
# 1
sudo useradd -c "Carol Lab" -s /bin/bash -e "$(date -d '+30 days' +%F)" labcarol

# 2
echo 'labcarol:First-Login-7731' | sudo chpasswd
sudo chage -d 0 labcarol
sudo chage -l labcarol

# 3
su - labcarol        # asks for her password, then forces a new one before giving a shell. Ctrl-C or exit.

# 4
sudo grep labcarol /etc/shadow | cut -d: -f2 | cut -c1-4
sudo usermod -L labcarol
sudo grep labcarol /etc/shadow | cut -d: -f2 | cut -c1-4      # now starts with !

# 5
sudo useradd -r -s /sbin/nologin -M labworker
id labworker         # below 1000: -r allocates from SYS_UID_MIN..SYS_UID_MAX, counting downward

# 6
sudo awk -F: '$2 !~ /^[!*]/ && $2 != "" {print $1}' /etc/shadow
```

</details>

---

## 3. Users and groups

A user has exactly one **primary group** (the GID in `/etc/passwd`), which becomes the group owner of files they create, and any number of **supplementary groups** (listed in `/etc/group`), which grant extra access.

Rocky uses **user private groups**: each user gets a primary group of the same name containing only them. That makes a group-writable umask safe and is what makes the shared-directory pattern in section 4 work.

```bash
id labcarol >/dev/null 2>&1 || sudo useradd labcarol     # created in the exercises above; make sure she exists
sudo groupadd labdevs
sudo groupadd -g 3000 labops                 # with a specific GID
sudo groupadd -r labdaemons                  # a system group
getent group labdevs labops

sudo usermod -aG labdevs labalice            # ADD labalice to labdevs
sudo usermod -aG labdevs,labops labcarol
id labalice; id labcarol
getent group labdevs
```

**`-aG`, never `-G` alone.** `usermod -G labdevs user` *replaces* the entire supplementary list with `labdevs`. Run against your own account without `-a`, it removes you from `wheel`, and with it your sudo. If there is no other admin and root login is disabled, that needs console access to repair.

Alternative that cannot make that mistake:

```bash
sudo gpasswd -a labalice labops              # add one user to one group
sudo gpasswd -d labalice labops              # remove one user from one group
sudo gpasswd -M labalice,labcarol labdevs    # set the complete member list
sudo groupmems -g labdevs -l                 # list members
```

### Membership takes effect at login

Group membership is read when a session starts. A running shell keeps the groups it started with.

```bash
sudo usermod -aG labdevs $USER
id -Gn                    # current shell: labdevs is NOT listed
id -Gn $USER              # the database: labdevs IS listed
newgrp labdevs            # start a subshell with labdevs active as the primary group
id -Gn; exit
# or log out and back in, or:  sudo -u $USER -i  /  su - $USER
```

Log out and back in now, so the rest of the lab sees your membership of `labdevs`.

### Groups that are really root

Some groups grant so much that membership equals root access. Review them like you review sudoers.

| Group | Why it matters |
|---|---|
| `wheel` | full sudo |
| `docker` | can talk to the Docker daemon, which runs as root. `docker run -v /:/host …` reads and writes every file on the machine |
| `disk` | raw read/write access to block devices |
| `adm`, `systemd-journal` | read system logs |

```bash
getent group wheel docker disk adm systemd-journal
```

### Modifying and deleting groups

```bash
sudo groupmod -n labplatform labops          # rename
sudo groupmod -g 3100 labplatform            # change the GID (files keep the OLD number and need a chgrp)
sudo groupdel labdaemons
sudo groupdel labalice                       # FAILS: it is a user's primary group
```

### Exercises 3

1. Create a group `labqa`. Add `labalice` and `labcarol` to it without disturbing their other groups. Verify from both directions (`id user` and `getent group`).
2. Remove `labcarol` from `labqa` again.
3. What is your own primary group? Create a file and confirm which group owns it.
4. Which human users on the tower are in `wheel`? In `docker`?
5. Demonstrate the `-G` mistake safely on `labcarol`: record her groups, run `usermod -G labqa labcarol`, and compare.

<details><summary>Solutions</summary>

```bash
# 1
sudo groupadd labqa
sudo usermod -aG labqa labalice; sudo usermod -aG labqa labcarol
id labalice; getent group labqa

# 2
sudo gpasswd -d labcarol labqa

# 3
id -gn; touch f; ls -l f

# 4
getent group wheel docker | cut -d: -f1,4

# 5
id -Gn labcarol                    # labcarol labdevs labplatform
sudo usermod -G labqa labcarol
id -Gn labcarol                    # labcarol labqa      ← labdevs and labplatform are gone
sudo usermod -aG labdevs,labplatform labcarol
```

</details>

---

## 4. Managing permissions

Lab 04 covered the mechanics of mode bits. This section is about designing access for more than one person.

### A shared project directory

Goal: members of `labdevs` can all create and edit files in `/srv/labshare`; nobody else can get in.

A naive attempt:

```bash
sudo mkdir /srv/labshare
sudo chgrp labdevs /srv/labshare
sudo chmod 770 /srv/labshare
sudo -u labalice touch /srv/labshare/alice.txt
ls -l /srv/labshare
```

`alice.txt` belongs to group `labalice`, her primary group, with mode 644. Carol is in `labdevs` and can list the directory, yet cannot edit the file. Two fixes are needed.

**Fix 1: setgid on the directory**, so new entries inherit the directory's group.

```bash
sudo chmod 2770 /srv/labshare                # or: chmod g+s
ls -ld /srv/labshare                         # drwxrws---
sudo -u labalice touch /srv/labshare/alice2.txt
ls -l /srv/labshare                          # group is now labdevs
```

**Fix 2: group-writable files.** The file is still `rw-r--r--` because the default umask is 022. Options: have the team use umask 002 (safe with user private groups), or use a default ACL (below), which does not depend on anyone's umask.

```bash
sudo -u labalice sh -c 'umask 002; touch /srv/labshare/alice3.txt'
ls -l /srv/labshare                          # rw-rw-r--
sudo -u labcarol sh -c 'echo "carol was here" >> /srv/labshare/alice3.txt' && echo "carol can write"
sudo -u labcarol sh -c 'echo "carol was here" >> /srv/labshare/alice2.txt' || echo "carol cannot write the 644 one"
```

**Optional: the sticky bit**, if members should not delete each other's files (the `/tmp` behaviour):

```bash
sudo chmod +t /srv/labshare                  # mode 3770, shown as drwxrws--T
sudo -u labcarol rm /srv/labshare/alice3.txt # Operation not permitted
sudo chmod -t /srv/labshare
```

### ACLs: when one owner and one group are not enough

A file has one owner and one group. To also give read access to one extra person, or a second group, use an **access control list**.

```bash
sudo useradd labdave                                  # not in labdevs
sudo -u labdave ls /srv/labshare                      # Permission denied

sudo setfacl -m u:labdave:rx /srv/labshare            # -m modify: user labdave gets r-x on the directory
sudo -u labdave ls /srv/labshare                      # works
getfacl /srv/labshare
ls -ld /srv/labshare                                  # a "+" after the mode marks the ACL
```

```
# file: srv/labshare
# owner: root
# group: labdevs
# flags: -s-
user::rwx
user:labdave:r-x
group::rwx
mask::rwx
other::---
```

| Entry | Meaning |
|---|---|
| `user::` | the owner |
| `user:NAME:` | a named user |
| `group::` | the owning group |
| `group:NAME:` | a named group |
| `mask::` | the upper limit applied to all named users, named groups and the owning group. `chmod g=` on an ACL'd file changes the mask |
| `other::` | everyone else |

```bash
sudo setfacl -m g:labqa:r-x /srv/labshare             # a second group
sudo setfacl -x u:labdave /srv/labshare               # -x remove one entry
sudo setfacl -R -m u:labdave:rX /srv/labshare         # -R recursive; capital X as in chmod
sudo setfacl -b /srv/labshare                         # -b remove all ACL entries
```

### Default ACLs: inheritance

A **default** ACL on a directory is copied onto everything created inside it. This solves the umask problem from Fix 2 properly.

```bash
sudo setfacl -d -m g:labdevs:rwx /srv/labshare        # -d: applies to new files and directories
sudo setfacl -d -m o::--- /srv/labshare
getfacl /srv/labshare                                 # note the default: lines
sudo -u labalice touch /srv/labshare/alice4.txt       # default umask 022 this time
getfacl /srv/labshare/alice4.txt                      # group:labdevs:rwx with an effective mask of rw-
sudo -u labcarol sh -c 'echo ok >> /srv/labshare/alice4.txt' && echo "carol can write, no umask change needed"
```

The complete recipe for a team directory:

```bash
# sudo install -d -o root -g TEAM -m 2770 /srv/DIR
# sudo setfacl -d -m g:TEAM:rwx,o::--- /srv/DIR
```

ACL caveats: plain `cp` and some backup tools drop them (`cp -a`, `rsync -A` and `tar --acls` keep them), and people forget they exist. Look for the `+` in `ls -l`.

### Delegating privileged commands

The same least-privilege thinking applies to sudo. Members of `labops` should be able to restart one service and read its logs, and nothing else.

```bash
sudo groupadd -f labops
sudo usermod -aG labops labalice
sudo visudo -f /etc/sudoers.d/91-labops
```

Enter:

```
Cmnd_Alias LAB_SVC = /usr/bin/systemctl restart crond, /usr/bin/systemctl status crond, /usr/bin/journalctl -u crond *
%labops ALL=(root) NOPASSWD: LAB_SVC
```

```bash
sudo visudo -c
sudo -l -U labalice                                   # what she may run
sudo -u labalice sudo -n /usr/bin/systemctl status crond | head -3      # allowed
sudo -u labalice sudo -n /usr/bin/systemctl restart sshd                # refused: "a password is required"
```

Things to know when writing such rules:

- The command must match exactly, arguments included. A `*` wildcard matches anything, including extra arguments you did not think of, so use it sparingly.
- Never delegate a command that can run other commands or edit arbitrary files: `vim`, `less`, `find`, `awk`, `tar`, `docker`, any interpreter. Each offers a way to a root shell. The site gtfobins.github.io catalogues them.
- Prefer a small wrapper script in `/usr/local/sbin` (root-owned, not writable by the group) that does exactly one thing, and delegate the script.

### Auditing access

Questions a review should answer, and the command for each:

```bash
awk -F: '$3 == 0 {print $1}' /etc/passwd                              # accounts with UID 0 (should be root only)
getent group wheel docker | cut -d: -f1,4                             # who is effectively root
sudo grep -rvE '^\s*(#|$)' /etc/sudoers /etc/sudoers.d/               # every active sudo rule
sudo awk -F: '$2 !~ /^[!*]/ && $2 != "" {print $1}' /etc/shadow       # accounts with a usable password
sudo find /home -name authorized_keys -exec ls -l {} + 2>/dev/null    # who can log in with a key
sudo find / -xdev -type d -perm -0002 ! -perm -1000 2>/dev/null       # world-writable dirs without sticky bit
sudo find / -xdev -type f -perm -4000 2>/dev/null                     # setuid programs
sudo getfacl -R -s -p /srv /home 2>/dev/null | grep '^# file'         # files that carry ACLs
```

### Exercises 4

1. Create `/srv/labdocs` so that `labdevs` members have full access, `labqa` members can read but not write, and everyone else is denied. New files must be group-writable for `labdevs` without anyone changing their umask.
2. Prove it: as `labalice` (in labdevs) create a file; as a `labqa`-only user read it and fail to modify it; as `labdave` fail to list the directory. You may need to adjust group memberships to get a QA-only user.
3. Explain what the `mask::` entry does. Set the mask on `/srv/labdocs` to `r-x` and observe the `#effective:` annotations. Restore it.
4. Write a sudoers rule allowing `labdave` to run `/usr/bin/dnf check-update` and nothing else, with a password. Verify with `sudo -l -U labdave`.
5. Why would allowing `labdave` to run `/usr/bin/dnf` (with any arguments) be equivalent to giving him root?

<details><summary>Solutions</summary>

```bash
# 1
sudo install -d -o root -g labdevs -m 2770 /srv/labdocs
sudo setfacl -m g:labqa:r-x /srv/labdocs
sudo setfacl -d -m g:labdevs:rwx,g:labqa:r-x,o::--- /srv/labdocs
getfacl /srv/labdocs

# 2
sudo useradd labeve && sudo usermod -aG labqa labeve
sudo -u labalice sh -c 'echo spec > /srv/labdocs/spec.txt'
sudo -u labeve cat /srv/labdocs/spec.txt                         # works
sudo -u labeve sh -c 'echo x >> /srv/labdocs/spec.txt'           # Permission denied
sudo -u labdave ls /srv/labdocs                                  # Permission denied

# 3
# The mask caps what named users, named groups and the owning group can actually get.
sudo setfacl -m m::r-x /srv/labdocs; getfacl /srv/labdocs        # group::rwx   #effective:r-x
sudo setfacl -m m::rwx /srv/labdocs

# 4
echo 'labdave ALL=(root) /usr/bin/dnf check-update' | sudo tee /etc/sudoers.d/92-labdave > /dev/null
sudo chmod 440 /etc/sudoers.d/92-labdave && sudo visudo -c
sudo -l -U labdave
# (tee is used here so the solution is one line; visudo -f is the safer habit.)

# 5
# dnf installs packages, and package install scripts run as root. He could build an RPM whose
# %post script does anything, and install it. The same goes for adding a repository he controls.
```

</details>

---

## Challenge: onboard and offboard

**Onboard** a contractor, `labfrank`, in as few commands as you can:

- full name "Frank Lab", bash shell, account expires in 14 days
- member of `labdevs`
- password must be changed at first login, and thereafter every 30 days
- an SSH public key installed for him, with correct ownership and modes (generate a throwaway key pair in `~/labwork/08` to use)

**Offboard** him:

- block all logins immediately, password and key
- kill his processes
- archive his home directory to `/root/offboard/labfrank-YYYY-MM-DD.tar.gz`
- find every file he owns outside his home directory
- remove the account and home directory, and confirm no ownerless files remain

<details><summary>Solution</summary>

```bash
# --- onboard ---
ssh-keygen -t ed25519 -N '' -f ~/labwork/08/frank_key -C labfrank
sudo useradd -c "Frank Lab" -s /bin/bash -G labdevs -e "$(date -d '+14 days' +%F)" labfrank
echo 'labfrank:Welcome-Change-Me-19' | sudo chpasswd
sudo chage -d 0 -M 30 labfrank
sudo install -d -o labfrank -g labfrank -m 700 /home/labfrank/.ssh
sudo install -o labfrank -g labfrank -m 600 ~/labwork/08/frank_key.pub /home/labfrank/.ssh/authorized_keys
sudo restorecon -Rv /home/labfrank/.ssh        # SELinux label; sshd ignores keys with the wrong one
sudo chage -l labfrank; id labfrank; sudo ls -la /home/labfrank/.ssh

# --- offboard ---
sudo usermod -L -e 1 -s /sbin/nologin labfrank
sudo pkill -KILL -u labfrank
sudo install -d -m 700 /root/offboard
sudo tar -czf "/root/offboard/labfrank-$(date +%F).tar.gz" -C /home labfrank
sudo find / -xdev -user labfrank ! -path '/home/labfrank/*' 2>/dev/null
sudo crontab -u labfrank -l 2>/dev/null
sudo userdel -r labfrank
sudo find / -xdev -nouser 2>/dev/null | head
```

On a machine managed with Ansible, both halves belong in a playbook using the `user`, `group` and `authorized_key` modules with `state: present` / `state: absent`. Doing it by hand once is how you learn what those modules do.

</details>

## Check yourself

<details><summary>What is the difference between <code>usermod -G docker walter</code> and <code>usermod -aG docker walter</code>?</summary>
Without <code>-a</code>, the supplementary group list is replaced: walter ends up in docker only and loses wheel and everything else. With <code>-a</code>, docker is appended to the existing list.
</details>

<details><summary>You added yourself to a group but still get "Permission denied". Why?</summary>
Group membership is read at login. The current session still has the old list. Log out and in, or use <code>newgrp</code>.
</details>

<details><summary>Does <code>usermod -L</code> stop a user logging in over SSH?</summary>
It disables password authentication only. A key in <code>authorized_keys</code> still works. Expire the account (<code>usermod -e 1</code>) or set the shell to nologin and remove the keys.
</details>

<details><summary>What does setgid on a directory do, and why does a shared directory need it?</summary>
New files and subdirectories inherit the directory's group rather than the creator's primary group. Without it each member's files belong to their private group and the others cannot write them.
</details>

<details><summary>Why is membership of the <code>docker</code> group equivalent to root?</summary>
The Docker daemon runs as root and does what its socket is told. A member can start a container with the host's root filesystem mounted and read or modify anything on it.
</details>

<details><summary>After <code>userdel bob</code> (no <code>-r</code>), <code>ls -l</code> shows a number where the owner should be. What is the risk?</summary>
The files still belong to bob's old UID. A future account assigned that UID automatically owns them.
</details>

## Cleanup

Run all of this. Check the final commands print nothing.

```bash
for u in labalice labbob labcarol labdave labeve labfrank labtemp labsvc labworker; do
  sudo pkill -KILL -u "$u" 2>/dev/null
  sudo userdel -r "$u" 2>/dev/null
done
for g in labdevs labops labplatform labqa labdaemons labalice labcarol; do
  sudo groupdel "$g" 2>/dev/null
done
sudo gpasswd -d "$USER" labdevs 2>/dev/null
sudo rm -f /etc/sudoers.d/91-labops /etc/sudoers.d/92-labdave
sudo rm -rf /srv/labshare /srv/labdocs /root/offboard /var/lib/labsvc /home/alice-new
sudo visudo -c
rm -rf ~/labwork/08

getent passwd | grep '^lab'      # expect nothing
getent group  | grep '^lab'      # expect nothing
id -Gn "$USER"                   # confirm you are still in wheel
```

Lab 09 recreates `labsvc` when it needs it.
