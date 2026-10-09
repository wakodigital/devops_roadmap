# Lab 04 — Working with Files

**Roadmap topics:** File Permissions · Archiving and Compressing · Copying and Renaming · Soft Links / Hard Links
**Time:** about 2.5 hours
**Touches system state:** no. Everything happens in `~/labwork/04`. A few commands use `sudo` to change ownership of lab files.

## What you will be able to do

- Read and set permissions in both symbolic and octal form, and predict what a given mode allows.
- Explain why "I have write permission on the file but cannot delete it" happens.
- Create, inspect and extract tar archives with any of the common compressors.
- Copy trees while preserving exactly the metadata you intend to preserve.
- Choose between a hard link and a symbolic link, and explain what each one is at the inode level.

## Setup

```bash
mkdir -p ~/labwork/04 && cd ~/labwork/04
```

---

## 1. File permissions

### The model

Every file has an **owner** (a user), a **group**, and a **mode**: three sets of three bits.

```
-rwxr-x---  1 walter devs  ... deploy.sh
 └┬┘└┬┘└┬┘
  │  │  └ other: everyone else            ---  nothing
  │  └ group: members of "devs"           r-x  read, execute
  └ user: the owner, walter               rwx  read, write, execute
```

The kernel checks them in order and stops at the first class that applies to you. If you are the owner, only the user bits count, even when the group or other bits are more generous.

What the bits mean depends on the type:

| Bit | On a file | On a directory |
|---|---|---|
| `r` | read the contents | list the names inside |
| `w` | change the contents | create, delete and rename entries inside |
| `x` | execute it as a program | enter it (`cd`) and reach things inside by name |

Consequences worth internalising:

- **Deleting a file is a write to its directory**, not to the file. You can delete a read-only file you do not own if you have `w` and `x` on the directory.
- A directory with `r` but no `x` lets you see names and nothing else. A directory with `x` but no `r` lets you open files whose names you already know.
- To reach `/a/b/c/file` you need `x` on `/`, `/a`, `/a/b` and `/a/b/c`.

### Octal

Each set is a 3-bit number: `r` = 4, `w` = 2, `x` = 1.

| Octal | Bits | | Octal | Bits |
|---|---|---|---|---|
| 7 | `rwx` | | 3 | `-wx` |
| 6 | `rw-` | | 2 | `-w-` |
| 5 | `r-x` | | 1 | `--x` |
| 4 | `r--` | | 0 | `---` |

| Mode | Typical use |
|---|---|
| `644` `rw-r--r--` | ordinary files, most of `/etc` |
| `600` `rw-------` | secrets: private keys, `.env` files, vault passwords |
| `755` `rwxr-xr-x` | programs, scripts, most directories |
| `700` `rwx------` | private directories: `~/.ssh` |
| `750` `rwxr-x---` | a directory shared with one group |
| `664` / `775` | group-writable files / directories |

### chmod

```bash
cd ~/labwork/04
touch report.txt script.sh
ls -l

chmod 600 report.txt          # octal: sets all nine bits at once
chmod 755 script.sh
ls -l

chmod u+x report.txt          # symbolic: change only what you name
chmod g+r,o-r report.txt
chmod go= report.txt          # set group and other to nothing
chmod a+r report.txt          # a = all three classes
chmod +x script.sh            # no class given = all, filtered by umask
ls -l

stat -c '%a %A %U:%G %n' *    # octal, symbolic, owner:group, name
```

Use octal when you know the full mode you want. Use symbolic when you want to change one bit and leave the rest alone.

Recursion needs care, because files and directories want different modes:

```bash
mkdir -p site/css && touch site/index.html site/css/main.css
chmod -R 644 site              # WRONG: directories lose x and become unenterable
ls site                        # may still list, but:
cat site/index.html            # Permission denied
chmod -R u+rwX,go+rX site      # capital X: add x only to directories (and files that already have x)
ls -lR site
```

`X` is the tool for "make this tree readable" without making every file executable. The alternative is `find`:

```bash
find site -type d -exec chmod 755 {} +
find site -type f -exec chmod 644 {} +
```

### chown and chgrp

Only root can give a file away to another user. You can change the group of your own file to any group you belong to.

```bash
id -nG                              # your groups
sudo chown root report.txt          # change owner
sudo chown root:root report.txt     # owner and group
sudo chown -R $USER:$USER site      # recursive
sudo chown $USER: report.txt        # trailing colon: user, and that user's login group
chgrp wheel report.txt              # works without sudo if you are in wheel
ls -l
```

### umask

New files start from `666` and new directories from `777`, and the **umask** removes bits.

```bash
umask               # 0022 is the usual default
umask -S            # the same, symbolic
touch new.txt; mkdir newdir; ls -ld new.txt newdir     # 644 and 755
umask 077
touch private.txt; mkdir privatedir; ls -ld private.txt privatedir    # 600 and 700
umask 022           # put it back
```

`umask 077` at the top of a script that writes secrets guarantees nothing it creates is readable by others, even briefly.

### Special bits

A fourth, leading octal digit holds three special bits.

| Bit | Octal | On a file | On a directory | Seen as |
|---|---|---|---|---|
| setuid | 4000 | runs with the **owner's** identity | (ignored) | `s` in the user x position |
| setgid | 2000 | runs with the file's group | new entries **inherit the directory's group** | `s` in the group x position |
| sticky | 1000 | (ignored) | only an entry's owner may delete or rename it | `t` in the other x position |

```bash
ls -l /usr/bin/passwd       # -rwsr-xr-x root: setuid, which is how you can change your own password
                            # even though /etc/shadow is writable only by root
ls -ld /tmp                 # drwxrwxrwt: sticky, so you cannot delete other users' temp files
find /usr/bin -perm -4000 -ls 2>/dev/null     # every setuid program: part of a security review
```

setgid on a directory is the basis of a shared team directory; you will build one in Lab 08.

```bash
mkdir shared
chmod 2775 shared           # or: chmod g+s shared
ls -ld shared               # drwxrwsr-x
```

### Beyond the nine bits on Rocky

When the mode looks right and access is still denied, two more layers exist:

- **SELinux**, enforcing by default on Rocky. Every file carries a label, visible with `ls -Z`. A process may only touch files whose label its policy permits, whatever the mode says. Quick check: `sudo ausearch -m AVC -ts recent` lists recent denials. Fix labels with `restorecon -Rv path`. Do not fix them by turning SELinux off.
- **ACLs**, extra per-user or per-group entries, shown by a `+` after the mode (Lab 08).

```bash
ls -Z ~/labwork/04 | head -3
ls -Zd /var/www /etc/ssh ~ 2>/dev/null
getenforce
```

### Exercises 1

1. Give the octal mode for each: `rwxr-x---`, `rw-rw-r--`, `r--------`, `rwx--x--x`.
2. Create `secret.txt` readable and writable only by you, using symbolic mode starting from whatever the default was.
3. Create a directory `dropbox` that other users can copy files **into** but cannot list. Which bits?
4. Make `locked.txt` with mode `444` inside a directory you own. Can you delete it? Try. Explain.
5. Create `d1/` containing `f1`. Run `chmod u-x d1`. What do `ls d1`, `ls -l d1` and `cat d1/f1` each do? Restore with `chmod u+x d1`.
6. Your private key was copied from another machine with mode `644`. What does `ssh` do, and what is the fix?

<details><summary>Solutions</summary>

```bash
# 1
# 750, 664, 400, 711

# 2
touch secret.txt && chmod go= secret.txt       # or chmod u=rw,go= secret.txt

# 3
mkdir dropbox && chmod 733 dropbox             # rwx-wx-wx: others have w+x but not r
# add the sticky bit (1733) so they cannot delete each other's files

# 4
touch locked.txt && chmod 444 locked.txt
rm locked.txt       # rm asks for confirmation because the file is write-protected, then succeeds.
                    # Deletion needs w+x on the directory, which you have.

# 5
mkdir d1 && touch d1/f1 && chmod u-x d1
ls d1               # prints f1 (r lets you read names), possibly with an error
ls -l d1            # names with ????????? where metadata should be: it cannot stat entries without x
cat d1/f1           # Permission denied
chmod u+x d1

# 6
# ssh refuses to use it: "UNPROTECTED PRIVATE KEY FILE ... permissions 0644 are too open".
chmod 600 ~/.ssh/id_ed25519
# and ~/.ssh itself should be 700
```

</details>

---

## 2. Archiving and compressing

Two separate jobs:

- **Archiving** combines many files and their metadata into one file. `tar` does this.
- **Compressing** makes one stream of bytes smaller. `gzip`, `bzip2`, `xz` and `zstd` do this, on a single file.

A `.tar.gz` is a tar archive that was then compressed with gzip. `tar` can call the compressor for you.

### tar

Three modes, one of which you always specify: **c**reate, e**x**tract, lis**t**. Then `f` and the archive name.

| Command | Meaning |
|---|---|
| `tar -cf a.tar dir/` | create |
| `tar -tf a.tar` | list contents |
| `tar -xf a.tar` | extract into the current directory |
| `tar -xf a.tar -C /dest` | extract into another directory |
| `tar -czf a.tar.gz dir/` | create with gzip |
| `tar -cjf a.tar.bz2 dir/` | create with bzip2 |
| `tar -cJf a.tar.xz dir/` | create with xz |
| `tar --zstd -cf a.tar.zst dir/` | create with zstd |
| `tar -caf a.tar.xz dir/` | `-a`: pick the compressor from the file extension |
| `tar -xf anything` | extraction auto-detects the compression |
| add `-v` | list each file as it is processed |
| add `-p` | preserve permissions on extract (the default for root) |

Build some data to work with:

```bash
cd ~/labwork/04
mkdir -p project/{src,docs,logs}
for i in $(seq 1 50); do echo "line $i of source code, repeated text compresses well" ; done > project/src/app.py
cp /etc/services project/docs/services.txt
head -c 2M /dev/urandom > project/logs/random.bin       # random data: will not compress
seq 1 200000 > project/logs/numbers.log                 # regular data: compresses a lot
du -sh project
```

```bash
tar -cf project.tar project/
tar -czf project.tar.gz project/
tar -cJf project.tar.xz project/
tar --zstd -cf project.tar.zst project/      # if this fails: sudo dnf install -y zstd
ls -lh project.tar*

tar -tvf project.tar.gz                      # look before you extract
tar -tf project.tar.gz | wc -l

mkdir restore
tar -xf project.tar.gz -C restore
diff -r project restore/project && echo "identical"

tar -xf project.tar.gz -C restore project/docs/services.txt     # extract one member
tar -czf src-only.tar.gz --exclude='logs' --exclude='*.bin' project/
tar -tf src-only.tar.gz
```

Habits:

- **List before extracting** an archive you did not make. A well-behaved archive contains one top-level directory. A badly made one (a "tarbomb") scatters hundreds of files into your current directory.
- tar stores paths as given. `tar -cf x.tar project/` stores `project/…`; `tar -cf x.tar /home/you/project` stores `home/you/project/…` (it strips the leading `/` and tells you). Use `-C` to control the stored prefix: `tar -czf x.tar.gz -C /home/you project`.
- To preserve ownership when restoring system files, create and extract as root.

### The compressors

| Tool | Extension | Speed | Ratio | Notes |
|---|---|---|---|---|
| `gzip` | `.gz` | fast | fair | universal; the default choice for compatibility |
| `bzip2` | `.bz2` | slow | better | mostly legacy |
| `xz` | `.xz` | slowest to compress | best | used for distro packages and kernel tarballs |
| `zstd` | `.zst` | very fast | near xz at high levels | the modern default; multithreaded with `-T0` |

```bash
cp project/logs/numbers.log .
ls -lh numbers.log
gzip numbers.log              # REPLACES numbers.log with numbers.log.gz
ls -lh numbers.log*
gunzip numbers.log.gz         # and back
gzip -k numbers.log           # -k keeps the original
gzip -9 -k -c numbers.log > numbers.max.gz     # -9 best, -1 fastest, -c write to stdout
xz -k numbers.log
ls -lh numbers.*

zcat numbers.log.gz | head -3        # read without decompressing to disk
zgrep -c 9999 numbers.log.gz         # grep inside
zless numbers.log.gz
```

`zcat`, `zgrep` and `zless` matter on servers because logrotate compresses old logs to `.gz`.

### zip

Zip archives and compresses in one step and is what Windows users expect.

```bash
sudo dnf install -y zip unzip
zip -r project.zip project/
unzip -l project.zip            # list
unzip -q project.zip -d unzipped/
```

Zip does not preserve Unix ownership and handles permissions and symlinks poorly. Use tar between Linux systems.

### Exercises 2

1. Compare the sizes of the four archives. Then compress `random.bin` and `numbers.log` individually with gzip and compare their ratios. Explain the difference.
2. Create an archive of `/etc/ssh` (you will need sudo) in `~/labwork/04/ssh-backup.tar.gz` such that the paths inside start with `ssh/`, not `etc/ssh/`.
3. List only the files in `project.tar.gz` whose names end in `.txt`, without extracting.
4. Time `gzip`, `xz` and `zstd` compressing a copy of `numbers.log`: `time gzip -k -c numbers.log > /dev/null`. Rank them.
5. Write the pipeline that tars `project/` and extracts it into `/tmp/copy/` in one line, with no intermediate file.

<details><summary>Solutions</summary>

```bash
# 1
ls -lh project.tar*
gzip -k -c project/logs/random.bin | wc -c      # about the same as the 2 MB input
gzip -k -c project/logs/numbers.log | wc -c     # a small fraction of the input
# Compression removes redundancy. Random data has none. Encrypted and already-compressed
# files (jpg, mp4, zip) behave like random data.

# 2
sudo tar -czf ~/labwork/04/ssh-backup.tar.gz -C /etc ssh
sudo tar -tzf ~/labwork/04/ssh-backup.tar.gz | head

# 3
tar -tf project.tar.gz | grep '\.txt$'
# or: tar -tf project.tar.gz --wildcards '*.txt'

# 4
time gzip -k -c numbers.log > /dev/null
time xz   -k -c numbers.log > /dev/null
time zstd -k -c numbers.log > /dev/null
# typically zstd < gzip < xz for time taken

# 5
mkdir -p /tmp/copy && tar -cf - project/ | tar -xf - -C /tmp/copy
# "-" as the filename means stdout when creating and stdin when extracting.
# Add ssh in the middle and this copies a tree between machines:
#   tar -czf - project/ | ssh host 'tar -xzf - -C /dest'
rm -r /tmp/copy
```

</details>

---

## 3. Copying and renaming

### cp

```bash
cd ~/labwork/04
cp report.txt report.bak             # file to file
cp report.txt project/docs/          # file into a directory
cp report.txt new.txt project/docs/  # several files into a directory
cp project project2                  # FAILS: -r not specified
cp -r project project2               # recursive
```

What gets preserved is the part that matters:

| Option | Effect |
|---|---|
| `-r` | recurse into directories |
| `-p` | preserve mode, ownership and timestamps |
| `-a` | archive: `-r`, preserve everything including symlinks as symlinks, SELinux context and extended attributes. **The right default for copying trees.** |
| `-i` | prompt before overwriting |
| `-n` | never overwrite |
| `-u` | copy only when the source is newer or the destination is missing |
| `-v` | report each file |
| `-l` / `-s` | make hard links / symlinks instead of copying |
| `--backup=numbered` | keep `file.~1~`, `file.~2~` when overwriting |

```bash
touch -d "2020-01-01" old.txt
cp old.txt copy-plain.txt
cp -p old.txt copy-preserved.txt
ls -l old.txt copy-plain.txt copy-preserved.txt     # compare the dates
```

A plain `cp` creates a new file: owned by you, timestamped now, mode filtered by umask. That is why a config backup should be `cp -a`, or the restored file may have the wrong owner and SELinux label.

The trailing-slash question with directories:

```bash
cp -a project backup1          # backup1 does not exist → it becomes a copy of project
cp -a project backup1          # backup1 now exists → you get backup1/project
cp -a project/. backup1/       # copy the CONTENTS of project into backup1, dotfiles included
```

### rsync for bigger jobs

`rsync` copies only what changed, can resume, and can delete extras at the destination. It works locally and over SSH with the same syntax (Lab 13).

```bash
sudo dnf install -y rsync
rsync -a project/ mirror/          # trailing slash on the source = "the contents of"
rsync -a project mirror2/          # no trailing slash = "the directory itself" → mirror2/project
echo "change" >> project/src/app.py
rsync -av project/ mirror/         # only app.py is sent
rsync -avn --delete project/ mirror/     # -n: dry run. Always dry-run before --delete.
```

### install: copy and set metadata in one step

```bash
install -D -m 755 script.sh ~/bin/myscript     # copy, set the mode, and create ~/bin if needed (-D)
install -d -m 750 newdir2                      # create a directory with a given mode
rm ~/bin/myscript
# typical real use:  sudo install -o root -g root -m 755 tool.sh /usr/local/bin/tool
```

### Renaming

One file: `mv old new` (Lab 01). Many files need a loop or a tool.

```bash
mkdir rn && cd rn
touch photo_{001..005}.jpeg report-2026-0{1..3}.TXT

# util-linux rename: rename <from> <to> <files>   (replaces the first occurrence in each name)
rename .jpeg .jpg *.jpeg
rename -v photo_ img- *.jpg
ls

# a loop with parameter expansion handles anything rename cannot
for f in *.TXT; do
  mv -v -- "$f" "${f%.TXT}.txt"       # ${f%.TXT} = $f with the trailing .TXT removed
done
ls
cd ..
```

Preview a rename loop by putting `echo` in front of `mv` first.

Debian and Ubuntu ship a different `rename` (a Perl script taking `s/from/to/` expressions). The loop works everywhere.

### Exercises 3

1. Back up `/etc/ssh/sshd_config` to `~/labwork/04/` so that the copy keeps the original's timestamps. Verify with `ls -l`. (Ownership cannot be kept without sudo. Why?)
2. Copy `project` to `project-v2`, then change a file in `project`. Use `diff -rq` to show which files differ.
3. Use `rsync` to make `project-v2` identical to `project` again. Dry-run first and read what it would do.
4. Create `a1.log` … `a5.log`. Rename them all to `a1.log.old` … `a5.log.old` with a loop.
5. Rename the files from exercise 4 to `2026-a1.log` … `2026-a5.log`.

<details><summary>Solutions</summary>

```bash
# 1
cp -p /etc/ssh/sshd_config .          # if unreadable to you: sudo cp -a, and the copy stays root-owned
ls -l /etc/ssh/sshd_config sshd_config
# Only root can create files owned by another user.

# 2
cp -a project project-v2
echo "# v2 change" >> project/src/app.py
diff -rq project project-v2

# 3
rsync -avn --delete project/ project-v2/
rsync -av  --delete project/ project-v2/
diff -rq project project-v2 && echo "in sync"

# 4
touch a{1..5}.log
for f in a?.log; do mv -- "$f" "$f.old"; done

# 5
for f in a?.log.old; do mv -- "$f" "2026-${f%.old}"; done
ls 2026-*
```

</details>

---

## 4. Soft links and hard links

### Inodes first

A file on disk is an **inode**: a numbered record holding the type, mode, owner, size, timestamps and the location of the data blocks. The inode does **not** hold a name.

A directory is a table mapping names to inode numbers. A "filename" is one row in such a table.

```bash
cd ~/labwork/04
echo "original content" > data.txt
ls -li data.txt             # first column: inode number. Third column: link count (1)
stat data.txt
```

### Hard links

A hard link is a second name for the same inode. Neither name is "the original". They are equals.

```bash
ln data.txt hard.txt
ls -li data.txt hard.txt    # same inode number, link count now 2
echo "appended via hard.txt" >> hard.txt
cat data.txt                # the change is visible: there is only one file
rm data.txt
cat hard.txt                # still there. Data is freed only when the link count reaches 0
                            # AND no process has the file open.
```

Limits: a hard link cannot cross filesystems (inode numbers are per-filesystem), and you cannot hard-link directories.

### Symbolic (soft) links

A symlink is a small separate file, with its own inode, whose content is a path. The kernel follows that path when you open the link.

```bash
echo "target content" > target.txt
ln -s target.txt soft.txt
ls -li target.txt soft.txt  # different inodes; "soft.txt -> target.txt"; type l; size = length of the path
cat soft.txt
readlink soft.txt           # the stored path
realpath soft.txt           # the fully resolved absolute path

rm target.txt
ls -l soft.txt              # still there, but dangling (often shown in red)
cat soft.txt                # No such file or directory
echo "recreated" > target.txt
cat soft.txt                # works again: the link stores a name, not an inode
```

Symlinks can point at directories and across filesystems. They can also break.

**Relative symlink targets are resolved from the directory containing the link**, not from where you ran `ln`:

```bash
mkdir -p app/releases/v1 app/releases/v2
ln -s releases/v1 app/current       # the text "releases/v1" is stored; resolved relative to app/
ls -l app/
ln -s app/releases/v1 app/broken    # stored text "app/releases/v1" → resolves to app/app/releases/v1
ls -l app/                          # broken is dangling
rm app/broken
```

When unsure, give an absolute target, or use `ln -sr` and let `ln` compute the relative path.

### Comparison

| | Hard link | Symbolic link |
|---|---|---|
| What it is | another directory entry for the same inode | a file containing a path |
| Own inode | no | yes |
| Cross filesystems | no | yes |
| Link to a directory | no | yes |
| If the target name is removed | data remains reachable | link dangles |
| Shown in `ls -l` | higher link count | `l` type and `-> target` |
| Typical use | snapshot-style backups, deduplication | version switching, config indirection, shortcuts |

### Where you meet them

```bash
ls -l /bin /sbin /lib                        # merged-/usr symlinks
ls -l /etc/localtime                         # timezone = a symlink into /usr/share/zoneinfo
ls -l /etc/systemd/system/multi-user.target.wants/ | head     # "enabling" a service creates a symlink (Lab 09)
ls -l /etc/alternatives/ | head              # the alternatives system: symlinks choosing a default
ls -l /proc/$$/cwd /proc/$$/exe              # even the kernel uses them
```

The **atomic release switch** is the pattern to know for deployments: each release is unpacked into its own directory, and a `current` symlink is repointed.

```bash
ln -sfn releases/v2 app/current     # -f replace, -n do not descend into the existing link
ls -l app/current
```

Rollback is the same command pointing back at `v1`.

### Finding links

```bash
find . -type l                          # all symlinks under here
find . -xtype l                         # dangling symlinks
find . -samefile hard.txt               # every name for the same inode
find /usr/bin -type f -links +1 | head  # files with more than one hard link
```

### Exercises 4

1. Create `file-a` with some text. Make a hard link `file-b` and a symlink `file-c` to it. Record the inode numbers and link counts of all three.
2. Delete `file-a`. Which of `file-b` and `file-c` still work? Why?
3. Recreate `file-a` with **different** text. What do `file-b` and `file-c` show now? Explain.
4. Try to hard-link a directory, and to hard-link `/etc/hostname` into `/dev/shm`. Read both error messages.
5. A directory's link count is 2 when empty. Create `d/` with three subdirectories and check its link count. Where do the links come from?
6. Set up `app/releases/v1` and `v2`, each containing a file `VERSION` with its number. Point `app/current` at v1, read `app/current/VERSION`, switch to v2, read again.

<details><summary>Solutions</summary>

```bash
# 1
echo "first" > file-a
ln file-a file-b
ln -s file-a file-c
ls -li file-a file-b file-c
# a and b share an inode with link count 2; c has its own inode and count 1

# 2
rm file-a
cat file-b        # "first": the inode still has one name
cat file-c        # No such file: the path stored in the symlink no longer exists

# 3
echo "second" > file-a
cat file-b        # "first": file-b is the old inode
cat file-c        # "second": the symlink follows the NAME file-a, now a brand new inode
ls -li file-a file-b

# 4
mkdir dd && ln dd dd2                 # hard link not allowed for directory
ln /etc/hostname /dev/shm/h           # Invalid cross-device link (different filesystems;
                                      # may instead be "Operation not permitted": the kernel's
                                      # protected_hardlinks setting forbids linking to files you cannot write)

# 5
mkdir -p d/{x,y,z} && stat -c '%h' d  # 5
# d's own name in its parent (1) + "." inside d (1) + ".." inside each of the 3 subdirectories (3)

# 6
mkdir -p app/releases/{v1,v2}
echo 1 > app/releases/v1/VERSION; echo 2 > app/releases/v2/VERSION
ln -sfn releases/v1 app/current; cat app/current/VERSION
ln -sfn releases/v2 app/current; cat app/current/VERSION
```

</details>

---

## Challenge: a backup script by hand

Do each step as a single command where possible.

1. Create `~/labwork/04/vault/` with mode `700`.
2. Archive `project/` into `vault/project-YYYY-MM-DD.tar.zst` using today's date from `$(date +%F)`, excluding `*.bin`.
3. Set the archive to mode `600`.
4. Create a symlink `vault/latest.tar.zst` pointing at it, with a relative target.
5. Verify the archive by listing it through the symlink.
6. Explain why a symlink is the right choice here and a hard link is not.

<details><summary>Solution</summary>

```bash
cd ~/labwork/04
install -d -m 700 vault
tar --zstd -cf "vault/project-$(date +%F).tar.zst" --exclude='*.bin' project/
chmod 600 vault/project-*.tar.zst
ln -sfn "project-$(date +%F).tar.zst" vault/latest.tar.zst
tar -tf vault/latest.tar.zst | head
# 6: "latest" should move to tomorrow's archive by repointing a name. A hard link would
# be a second name for today's data, and deleting today's file would not free the space.
```

</details>

## Check yourself

<details><summary>A file is mode 600 and owned by root. It sits in a directory you own. Can you read it? Delete it?</summary>
You cannot read it. You can delete it, because deletion is a write to the directory, which you control.
</details>

<details><summary>What does <code>chmod -R 755 .</code> do wrong on a source tree, and what is the better command?</summary>
It makes every file executable. <code>chmod -R u=rwX,go=rX .</code> gives directories their x bit and leaves plain files non-executable.
</details>

<details><summary>What is the practical difference between <code>cp -r</code> and <code>cp -a</code>?</summary>
<code>-a</code> also preserves ownership (when root), mode, timestamps, symlinks as symlinks, and SELinux contexts. <code>-r</code> alone creates new files owned by you, with new timestamps and modes filtered by your umask.
</details>

<details><summary>You delete a 10 GB log file but <code>df</code> shows no space freed. Why?</summary>
A process still has it open. The inode's link count is 0 but the data is kept until the last file descriptor closes. Find it with <code>sudo lsof +L1</code>, then restart or signal the process.
</details>

<details><summary>Why list a tarball before extracting it?</summary>
To see whether it unpacks into a single directory or sprays files into the current one, and to see whether any member paths are absolute or contain <code>..</code>.
</details>

## Cleanup

```bash
cd ~ && sudo rm -rf ~/labwork/04       # sudo because a few lab files were chowned to root
```
