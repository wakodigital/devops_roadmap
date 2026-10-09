# Lab 10 — Package Management

**Roadmap topics:** Package Repositories · Finding & Installing Packages · Listing Installed Packages · Install / Remove / Upgrade Packages · Snap
**Time:** about 2.5 hours
**Touches system state:** installs and removes a few small packages (`zsh`, `mc`, `dnf-utils`, the versionlock plugin). It does **not** run a full system upgrade; section 4 explains why that is a planned event on your tower. The Snap section is optional and clearly marked.

## What you will be able to do

- Explain what a package and a repository are, and where dnf gets its trust from.
- Find which package provides a command or file, before and after it is installed.
- Answer "what is installed, when, from where, and which package owns this file".
- Install, remove, upgrade, downgrade, pin and roll back packages.
- Plan an upgrade on a machine that runs services.
- Translate everything to `apt` when you are on Debian or Ubuntu.

## Background

A **package** is an archive of files plus metadata: name, version, dependencies, and scripts to run on install and removal. Rocky uses the **RPM** format.

Two layers of tooling:

| Layer | Tool | Job |
|---|---|---|
| low | `rpm` | install, remove, query and verify individual `.rpm` files. Knows nothing about repositories and does not resolve dependencies |
| high | `dnf` | find packages in repositories, resolve dependencies, download, verify signatures, then drive `rpm`. Keeps a transaction history |

Use `dnf` to change the system and `rpm -q…` to interrogate it. `yum` on Rocky 9 is a symlink to `dnf`.

A package name encodes its identity:

```
openssh-server-8.7p1-43.el9.x86_64
└─────┬──────┘ └─┬─┘ └──┬──┘ └──┬─┘
     name     version release  architecture   (noarch = any)
```

`el9` marks a build for Enterprise Linux 9. The version stays fixed for the life of the release; security fixes are **backported** and only the release number increases. That is why a scanner that compares upstream version numbers often reports false positives on RHEL-family systems.

## Setup

```bash
mkdir -p ~/labwork/10 && cd ~/labwork/10
sudo dnf install -y dnf-utils          # adds repoquery helpers, needs-restarting, dnf download, config-manager
```

---

## 1. Package repositories

A **repository** is a directory of packages on a web server, plus an index (`repodata`) describing them. dnf downloads the indexes, works out what to install, and fetches the packages.

```bash
dnf repolist                           # enabled repositories
dnf repolist --all                     # including disabled ones
dnf repolist -v | less                 # URL, package count, size and last update of each
dnf repoinfo baseos
```

Rocky 9's own repositories:

| Repo id | Contents | Default |
|---|---|---|
| `baseos` | the core operating system | enabled |
| `appstream` | applications, languages, databases | enabled |
| `extras` | small extras, including the release packages that set up other repos | enabled |
| `crb` | CodeReady Builder: development headers and build tools. Many EPEL packages depend on it | disabled |
| `highavailability`, `resilientstorage`, `nfv`, `rt`, `sap`, `saphana`, `plus`, `devel` | specialised sets | disabled |

Anything else in your list was added later. On your tower expect at least a Docker repository.

### Where repositories are defined

```bash
ls /etc/yum.repos.d/
sed -n '1,25p' /etc/yum.repos.d/rocky.repo
```

One stanza:

```ini
[baseos]
name=Rocky Linux $releasever - BaseOS
mirrorlist=https://mirrors.rockylinux.org/mirrorlist?arch=$basearch&repo=BaseOS-$releasever$rltype
#baseurl=http://dl.rockylinux.org/$contentdir/$releasever/BaseOS/$basearch/os/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-Rocky-9
```

| Key | Meaning |
|---|---|
| `[id]` | the repo's id, used in commands |
| `baseurl=` or `mirrorlist=` / `metalink=` | one fixed server, or a service that returns a list of mirrors |
| `enabled=` | 1 or 0 |
| `gpgcheck=1` | refuse packages whose signature does not verify. **Never set this to 0** |
| `gpgkey=` | the public key the packages must be signed with |
| `priority=` | lower number wins when two repos carry the same package |
| `exclude=` / `includepkgs=` | hide or allow-list packages from this repo |
| `$releasever`, `$basearch` | substituted with `9` and `x86_64` |

dnf's own settings are in `/etc/dnf/dnf.conf`:

```bash
cat /etc/dnf/dnf.conf
```

`installonly_limit=3` there is why you have up to three kernels installed side by side: kernels are installed alongside, never replaced, so the previous one remains bootable.

### Trust

Installing a package runs its scripts as root. The protection is the signature check: each repository's packages are signed, and you choose which signing keys to trust when you add the repository.

```bash
rpm -qa 'gpg-pubkey*'                                   # keys imported into the RPM database
rpm -qa 'gpg-pubkey*' --qf '%{NAME}-%{VERSION}  %{SUMMARY}\n'
ls /etc/pki/rpm-gpg/
rpm -qi openssh-server | grep -E 'Signature|Vendor|Packager'
rpm -K /var/cache/dnf/*/packages/*.rpm 2>/dev/null | head -3     # verify cached package signatures, if any are cached
```

Adding a repository means trusting its owner with root on your machine, permanently, including for future updates of anything they choose to publish. Add few, and prefer official ones.

### Adding and toggling repositories

```bash
dnf repolist --all | grep -i crb                        # is CRB enabled or disabled right now?
dnf --enablerepo=crb info ninja-build                   # enable a repo for ONE command only
dnf --disablerepo='*' --enablerepo=baseos list available | head
# sudo dnf config-manager --set-enabled crb             # enable it permanently (edits the .repo file)
# sudo dnf config-manager --set-disabled crb            # and disable it again
```

**EPEL** (Extra Packages for Enterprise Linux), maintained by the Fedora project, is the standard extra repository: `htop`, `ncdu`, `tldr`, `shellcheck`, `fail2ban` and several thousand more. It is installed by a package that drops a `.repo` file and a key:

```bash
dnf info epel-release                                   # it comes from the "extras" repo
# sudo dnf install -y epel-release                      # your decision: later labs mention EPEL tools as optional
# sudo dnf config-manager --set-enabled crb             # EPEL's recommended companion
```

Third-party vendors publish a `.repo` file to add directly:

```bash
# sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo   # how Docker's repo is usually added
sed -n '1,8p' /etc/yum.repos.d/docker-ce.repo 2>/dev/null
```

If your Ansible playbook set up Docker, look at how it did this: probably the `yum_repository` or `get_url` module followed by `dnf`. Reading the repo file here shows what those tasks produce.

### Metadata cache

```bash
sudo dnf makecache                    # download fresh indexes now
sudo dnf clean all                    # delete cached indexes and packages
sudo du -sh /var/cache/dnf
dnf -C list installed | head -3       # -C: use the cache only, do not touch the network
```

When dnf reports a package as missing that you know exists, stale metadata is the first suspect: `sudo dnf clean all && sudo dnf makecache`.

### Module streams

Some AppStream software is offered in several major versions at once, as **module streams**.

```bash
dnf module list 2>/dev/null | head -30
dnf module list nodejs postgresql 2>/dev/null
# sudo dnf module enable nodejs:20 && sudo dnf install nodejs     # choose a stream, then install
```

A `[d]` marks the default stream, `[e]` an enabled one. Only one stream of a module can be active at a time.

### Exercises 1

1. List every repository enabled on the tower, with the number of packages each provides.
2. Which repositories are **not** from Rocky? For each, find its `.repo` file, its base URL and its GPG key.
3. Is `gpgcheck` enabled in every repo stanza on the system? Write one command that proves it.
4. How many packages are available in `appstream`? In `crb` (without enabling it permanently)?
5. Which repository would `htop` come from? Is that repository enabled?

<details><summary>Solutions</summary>

```bash
# 1
dnf repolist -v | grep -E '^Repo-(id|pkgs)'

# 2
grep -L -i rocky /etc/yum.repos.d/*.repo
grep -E '^\[|^baseurl|^gpgkey|^enabled' /etc/yum.repos.d/docker-ce.repo | head

# 3
grep -H '^gpgcheck' /etc/yum.repos.d/*.repo | grep -v '=1' || echo "all stanzas with a gpgcheck line have it on"
# (a stanza with no gpgcheck line inherits gpgcheck=1 from /etc/dnf/dnf.conf)

# 4
dnf repoquery --repo appstream -q | wc -l
dnf repoquery --repo crb -q | wc -l

# 5
dnf --enablerepo='*' info htop 2>/dev/null | grep -E '^Repo'      # only resolves if EPEL is defined
# htop is in EPEL. If EPEL is not installed, dnf simply reports no match.
```

</details>

---

## 2. Finding packages

```bash
dnf search zsh                         # names and summaries
dnf search --all "z shell"             # descriptions too
dnf info zsh                           # version, repo, size, licence, description
dnf list zsh                           # one line: installed or available, and from where
dnf list 'python3-*' | head            # glob
dnf list available 'nmap*'
dnf list --showduplicates kernel       # every version the repos offer
```

### Which package provides this command or file?

This is the lookup you will use most: you know the command you want, not the package name.

```bash
dnf provides dig                       # a command name
dnf provides '*/bin/dig'               # a path pattern (quote it)
dnf provides '*/semanage'
dnf provides /etc/ssh/sshd_config
dnf provides 'libssl.so.3()(64bit)'    # a shared library
```

`dig` is in `bind-utils`, `semanage` in `policycoreutils-python-utils`, `netstat` in `net-tools`, `ss` and `ip` in `iproute`. Package names frequently differ from command names.

### Inspecting a package you have not installed

```bash
dnf repoquery -l zsh | head -20        # files it would install
dnf repoquery --requires zsh           # what it needs
dnf repoquery --requires --resolve zsh # ...as package names
dnf repoquery --whatrequires zsh       # what needs it
dnf repoquery -i zsh                   # info
dnf deplist zsh | head -20             # each dependency and what satisfies it
dnf download zsh                       # fetch the .rpm into the current directory without installing
rpm -qpl zsh-*.rpm | head              # -p: query a package FILE instead of the database
rpm -qpi zsh-*.rpm
rpm -qp --scripts zsh-*.rpm            # the scripts it would run as root
```

Reading `--scripts` before installing a package from an unfamiliar source is the equivalent of reading a shell script before piping it to `bash`.

### Groups

```bash
dnf group list
dnf group list --hidden | head -40
dnf group info "Development Tools"     # compilers, make, git: one command to install a toolchain
# sudo dnf group install "Development Tools"
```

### Exercises 2

1. Which package provides each of these commands: `traceroute`, `nslookup`, `strace`, `htpasswd`, `killall`?
2. How large is `nmap` to download, and how large when installed?
3. List the configuration files the `chrony` package ships, without relying on it being installed.
4. Which packages would be pulled in if you installed `mc`? (Dry run.)
5. Download the `tree` rpm and list its contents and install scripts.

<details><summary>Solutions</summary>

```bash
# 1
for c in traceroute nslookup strace htpasswd killall; do
  printf '%-12s ' "$c"; dnf provides -q "*/bin/$c" 2>/dev/null | head -1
done
# traceroute, bind-utils, strace, httpd-tools, psmisc

# 2
dnf info nmap | grep -E '^(Size|Installed size|Download size)'
# "Size" means download size for an available package and installed size for an installed one.

# 3
dnf repoquery -l chrony | grep '^/etc'

# 4
dnf install --assumeno mc              # shows the transaction, then answers "no"

# 5
dnf download tree && rpm -qpl tree-*.rpm && rpm -qp --scripts tree-*.rpm
```

</details>

---

## 3. Listing installed packages

```bash
dnf list installed | head
dnf list installed | wc -l
dnf list installed 'kernel*'
rpm -qa | wc -l
rpm -qa | sort | head
rpm -qa 'openssh*'
rpm -q bash                            # is it installed, and which version?
rpm -q nosuchpackage; echo "exit: $?"  # non-zero when not installed: usable in scripts
```

The `@repo` column in `dnf list installed` records where each package came from. `@System` or `@anaconda` means it came with the installer. `@@commandline` means a local `.rpm` file.

### rpm queries

`rpm -q` plus one letter. These are worth memorising.

| Query | Answers |
|---|---|
| `rpm -qi PKG` | **i**nformation: version, install date, size, vendor, description |
| `rpm -ql PKG` | **l**ist of files |
| `rpm -qc PKG` | **c**onfiguration files only |
| `rpm -qd PKG` | **d**ocumentation files only |
| `rpm -qf /path` | which package owns this **f**ile |
| `rpm -qR PKG` | what it **R**equires |
| `rpm -q --whatrequires PKG` | what requires it |
| `rpm -q --whatprovides X` | what provides capability X |
| `rpm -q --scripts PKG` | install and removal scripts |
| `rpm -q --changelog PKG` | packaging changelog, with CVE numbers for security fixes |
| `rpm -qa --last` | every package, most recently installed first |

```bash
rpm -qi openssh-server
rpm -qc openssh-server                 # where is its configuration?
rpm -ql openssh-server | grep bin      # what commands did it install?
rpm -qf /usr/bin/ss                    # iproute
rpm -qf "$(command -v dig)" 2>/dev/null
rpm -qf /etc/passwd                    # setup
rpm -q --changelog openssh-server | head -15
rpm -qa --last | head -15              # what changed most recently on this machine
```

`rpm -qf` on a file that prints `not owned by any package` tells you the file was created by hand, by a script, or by software installed outside the package manager.

### Custom output

```bash
rpm -qa --qf '%{NAME}\t%{VERSION}-%{RELEASE}\t%{VENDOR}\n' | sort | head
rpm -qa --qf '%{SIZE}\t%{NAME}\n' | sort -rn | head -10 | awk '{printf "%7.1f MB  %s\n", $1 / 1048576, $2}'
rpm -qa --qf '%{INSTALLTIME:date}  %{NAME}\n' | head -3
rpm -qa --qf '%{VENDOR}\n' | sort | uniq -c | sort -rn      # who built what is on this machine
rpm --querytags | head -20                                  # every available field
```

The vendor count is a quick supply-chain summary: nearly everything should say Rocky, with a few from Docker and whatever else you chose to add.

### What did *I* install?

```bash
dnf repoquery --userinstalled | head -30      # packages installed on request, as opposed to as dependencies
dnf history                                   # every dnf transaction: id, command line, date, action, count
dnf history info last                         # full detail of the most recent one
dnf history list zsh                          # transactions that touched a package
sudo tail -20 /var/log/dnf.rpm.log            # the plain log of installs, upgrades and removals
```

### Verifying installed files

`rpm -V` compares files on disk against what the package installed.

```bash
rpm -V openssh-server                  # no output = unchanged
sudo rpm -Va 2>/dev/null | head -20    # the whole system (slow)
```

```
S.5....T.  c /etc/ssh/sshd_config
```

| Character | Changed attribute |
|---|---|
| `S` | size |
| `5` | checksum (content) |
| `T` | modification time |
| `M` | mode |
| `U` / `G` | owner / group |
| `c` | the file is marked as configuration |
| `missing` | the file is gone |

Changed **config** files are expected. A changed **binary** under `/usr/bin` or `/usr/sbin` is not, and is a classic indicator of compromise or disk corruption.

### Exercises 3

1. How many packages are installed? How many of them were installed in the last 7 days?
2. Which package owns `/usr/bin/systemctl`? `/etc/fstab`? `/usr/local/bin/server-review` (if you created it)?
3. Which three installed packages take the most disk space?
4. List every config file from installed packages that has been modified since installation. (Hint: `rpm -Va`, then filter for the `c` marker and a `5`.)
5. What did the most recent `dnf` transaction on the tower do, and who ran it?
6. Which packages on the tower were not built by Rocky?

<details><summary>Solutions</summary>

```bash
# 1
rpm -qa | wc -l
rpm -qa --qf '%{INSTALLTIME}\n' | awk -v c="$(date -d '7 days ago' +%s)" '$1 > c' | wc -l

# 2
rpm -qf /usr/bin/systemctl /etc/fstab /usr/local/bin/server-review
# systemd, setup, "not owned by any package"

# 3
rpm -qa --qf '%{SIZE} %{NAME}\n' | sort -rn | head -3

# 4
sudo rpm -Va 2>/dev/null | awk '$2 == "c" && $1 ~ /5/ {print $3}'

# 5
sudo dnf history info last | head -15       # "User", "Command Line", "Packages Altered"

# 6
rpm -qa --qf '%{VENDOR}\t%{NAME}\n' | grep -vi '^rocky' | sort
```

</details>

---

## 4. Install, remove, upgrade

### Install

```bash
sudo dnf install zsh                   # shows the transaction and asks
sudo dnf install -y mc                 # -y answers yes; note the dependencies listed separately
dnf history info last                  # what that transaction did
rpm -q zsh mc
```

Read the summary before answering `y`. It lists what will be **installed**, **upgraded** and, the line that deserves attention, **removed**.

```bash
dnf download words                     # a small word-list package, fetched as a file
sudo dnf install -y ./words-*.rpm      # install a local file: dnf still resolves its dependencies from the repos
sudo dnf reinstall -y words            # put back a package's files (repairs deleted or damaged files)
sudo dnf install --assumeno nmap       # dry run
sudo dnf install --downloadonly nmap   # fetch into the cache without installing
```

Prefer `dnf install ./file.rpm` to `rpm -i file.rpm`: `rpm` fails on missing dependencies and leaves you to chase them.

### Remove

```bash
sudo dnf remove mc                     # also offers to remove dependencies nothing else needs
sudo dnf autoremove                    # remove leftover dependencies. READ the list before agreeing
dnf repoquery --unneeded               # what autoremove considers removable
```

Removing a package does not delete configuration files you modified (they are kept, sometimes as `.rpmsave`) or data the software created (`/var/lib/…`). Clean those up by hand when you mean to.

`dnf remove` follows reverse dependencies: removing a library removes everything that needs it. If the removal list is longer than you expected, answer `n` and find out why.

### Upgrade

```bash
dnf check-update                       # list available updates. Exit status 100 = updates exist, 0 = none
echo "exit: $?"
dnf check-update | wc -l
dnf updateinfo                         # summary by type: security, bugfix, enhancement
dnf updateinfo list --security         # advisories, with severity
dnf updateinfo info --security | less  # CVE numbers and descriptions
```

```bash
sudo dnf upgrade zsh                   # one package
# sudo dnf upgrade --security          # only packages with security advisories
# sudo dnf upgrade                     # everything
```

**Plan a full upgrade on the tower; do not run it as a lab step.** Things a blanket `dnf upgrade` can do to a machine that hosts services:

| Update to | Consequence |
|---|---|
| `docker-ce`, `containerd.io` | the Docker daemon restarts, and with it every container, unless `live-restore` is configured |
| `kernel` | takes effect only after a reboot |
| `glibc`, `openssl`, `systemd` | running services keep the old library in memory until they are restarted |
| `openssh-server` | `sshd` restarts; existing sessions survive, but a broken config would block new ones |
| `NetworkManager`, Wi-Fi firmware | a brief network drop is possible, which on a Wi-Fi-only headless machine deserves respect |

A sensible routine:

```bash
dnf check-update                               # 1. see what is coming
sudo dnf upgrade --assumeno | tail -30         # 2. read the transaction
# 3. pick a quiet time; make sure you have a way back in (console, or at least a second SSH session)
# 4. sudo dnf upgrade
sudo dnf needs-restarting -r                   # 5. is a reboot required? (exit status 1 = yes)
sudo dnf needs-restarting -s                   # 6. which services are still running old code?
# 7. restart those services, or reboot, and run your server-review afterwards
```

### Holding a package back

```bash
sudo dnf install -y python3-dnf-plugin-versionlock
sudo dnf versionlock add zsh           # pin the installed version
dnf versionlock list
sudo dnf versionlock delete zsh
sudo dnf upgrade --exclude='docker-ce*' --assumeno | tail -5     # skip something for one run
```

Pinning a database or container runtime, and upgrading it deliberately, is common practice. Remember that a pinned package also stops receiving security fixes.

### Undoing

Every dnf transaction can be reversed, as long as the old packages are still obtainable.

```bash
dnf history | head
sudo dnf install -y mc
dnf history info last
sudo dnf history undo last -y          # removes mc and the dependencies that came with it
rpm -q mc
# sudo dnf history rollback 42         # return to the state just after transaction 42
# sudo dnf downgrade PKG               # step one package back a version, if the repo still has it
```

### Unattended updates

```bash
dnf info dnf-automatic
# sudo dnf install -y dnf-automatic
# /etc/dnf/automatic.conf:  upgrade_type = security   apply_updates = yes
# sudo systemctl enable --now dnf-automatic.timer
```

Automatic security updates are the right default for a machine reachable from the internet. Combine them with a version lock on anything whose restart you want to control, and note that it is a systemd timer like the one you built in Lab 09.

### On Debian and Ubuntu

| Task | dnf / rpm | apt / dpkg |
|---|---|---|
| Refresh indexes | `dnf makecache` (automatic) | `apt update` (**required** before installing) |
| Search | `dnf search X` | `apt search X` |
| Info | `dnf info X` | `apt show X` |
| Install | `dnf install X` | `apt install X` |
| Install a local file | `dnf install ./x.rpm` | `apt install ./x.deb` |
| Remove | `dnf remove X` | `apt remove X` (`apt purge X` also deletes config) |
| Remove unused deps | `dnf autoremove` | `apt autoremove` |
| Upgrade all | `dnf upgrade` | `apt upgrade` / `apt full-upgrade` |
| List installed | `dnf list installed`, `rpm -qa` | `apt list --installed`, `dpkg -l` |
| Files in a package | `rpm -ql X` | `dpkg -L X` |
| Owner of a file | `rpm -qf /path` | `dpkg -S /path` |
| Which package has a file | `dnf provides '*/f'` | `apt-file search f` |
| Hold a version | `dnf versionlock add X` | `apt-mark hold X` |
| Repo definitions | `/etc/yum.repos.d/*.repo` | `/etc/apt/sources.list.d/` |
| History | `dnf history` | `/var/log/apt/history.log` |

In a Dockerfile you will write the Debian side constantly: `RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*`. The Rocky equivalent is `RUN dnf install -y curl && dnf clean all`.

### Exercises 4

1. Install `zsh`. Find its config files, its binary, and the transaction ID that installed it. Then undo that transaction and confirm it is gone.
2. How many updates are pending on the tower? How many are security updates? Is a reboot currently required?
3. Would a full upgrade touch `docker-ce` or the kernel right now? Find out without changing anything.
4. Delete `/usr/bin/tree` by hand (`sudo rm`). Show that `rpm -V tree` notices. Repair it without removing and installing.
5. Lock the version of `tree`, show the lock, and remove the lock.
6. Write a one-line health check suitable for a monitoring script: print `SECURITY UPDATES: N` where N is the number of pending security advisories.

<details><summary>Solutions</summary>

```bash
# 1
sudo dnf install -y zsh
rpm -qc zsh; rpm -ql zsh | grep '/bin/'
dnf history list zsh
sudo dnf history undo last -y; rpm -q zsh

# 2
dnf check-update -q | grep -c .
dnf updateinfo list --security -q | grep -c .
sudo dnf needs-restarting -r

# 3
sudo dnf upgrade --assumeno | grep -Ei 'docker|containerd|kernel'

# 4
sudo rm /usr/bin/tree
rpm -V tree                      # missing     /usr/bin/tree
sudo dnf reinstall -y tree
rpm -V tree && echo "clean"

# 5
sudo dnf versionlock add tree; dnf versionlock list; sudo dnf versionlock delete tree

# 6
echo "SECURITY UPDATES: $(dnf updateinfo list --security -q 2>/dev/null | grep -c .)"
```

</details>

---

## 5. Snap (optional on Rocky)

**Snap** is Canonical's universal package format. A snap bundles an application together with its libraries into one compressed, read-only image, runs it in a sandbox, and updates it automatically from the Snap Store. It is installed by default on Ubuntu, where you will meet it constantly. On Rocky it is available from EPEL and is rarely used.

How it differs from RPM packages:

| | RPM via dnf | Snap |
|---|---|---|
| Dependencies | shared with the system | bundled in the package |
| Same package across distros | no | yes |
| Updates | when you run them | automatic, several checks a day (can be scheduled or held) |
| Rollback | `dnf history undo` | `snap revert` to the previous revision, kept on disk |
| Isolation | none by default | sandboxed ("strict" confinement), with declared interfaces |
| Disk and start-up cost | lower | higher: each snap is a mounted squashfs image |
| Source | repositories you choose | the Snap Store, a single source run by Canonical |
| Release tracks | repos | channels: `stable`, `candidate`, `beta`, `edge`, plus tracks such as `22/stable` |

**Flatpak** is the comparable format favoured by the Fedora and RHEL world, aimed at desktop applications. On a RHEL-family **server**, the usual answer to "I need a newer or self-contained version of X" is a container, not a snap or a flatpak.

### Commands to recognise

```bash
snap find hello                  # search the store
snap info hello-world            # channels, versions, publisher, confinement
sudo snap install hello-world    # install from the stable channel
sudo snap install --channel=edge NAME
sudo snap install --classic NAME # "classic" confinement: no sandbox (editors, compilers)
snap list                        # installed snaps, with version, revision and channel
snap list --all                  # including older revisions kept for rollback
sudo snap refresh                # update everything now
sudo snap refresh --hold=72h     # postpone automatic updates
snap refresh --time              # the update schedule
sudo snap revert NAME            # go back to the previous revision
snap connections NAME            # which system interfaces the sandbox was granted
sudo snap remove --purge NAME
snap changes                     # history of snap operations
```

### Trying it on the tower

Read this before deciding. Installing snapd adds: the EPEL repository, a permanently running `snapd` daemon that updates software by itself, an SELinux policy module, and one loop-mounted filesystem per snap revision (these appear in `lsblk`, `df` and `mount`, which is noise in Lab 11). None of this is harmful and all of it is removable. If you would rather not, read the commands above and move on; nothing later depends on snap.

<details><summary>Install, use and completely remove snapd on Rocky 9</summary>

```bash
# install
sudo dnf install -y epel-release
sudo dnf install -y snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap          # needed for classic snaps
# log out and back in so /var/lib/snapd/snap/bin is on your PATH; the first use may need a minute

# use
snap version
sudo snap install hello-world
hello-world
snap list
snap info hello-world
lsblk | grep loop                             # each snap is a mounted squashfs image
mount | grep snap
systemctl list-units 'snap*'
ls /var/lib/snapd/snaps/

# remove everything
sudo snap remove --purge hello-world
sudo snap list 2>/dev/null                    # core snaps may remain; remove each: sudo snap remove --purge NAME
sudo systemctl disable --now snapd.socket snapd.service
sudo dnf remove -y snapd
sudo rm -rf /snap /var/lib/snapd /var/snap ~/snap
sudo dnf history list epel-release            # decide whether to keep EPEL
# sudo dnf remove -y epel-release
```

</details>

### Exercises 5

1. Give two reasons a team might choose a snap over the distribution's own package, and two reasons a server administrator might avoid snaps.
2. On an Ubuntu server, `lsblk` shows a dozen `loop` devices mounted under `/snap`. What are they?
3. What is the difference between strict and classic confinement?

<details><summary>Solutions</summary>

1. For: the same package works on every distribution; it carries its own dependencies, so a newer version than the distro ships can be installed without conflicts; automatic updates with one-step rollback. Against: updates happen on the store's schedule, which conflicts with change control; a single vendor-run store is the only source; extra daemon, disk use and slower start-up; on RHEL-family systems it is outside the supported, SELinux-integrated packaging.
2. Each is one revision of one installed snap: a read-only squashfs image mounted through a loop device.
3. Strict snaps are sandboxed and reach the system only through declared interfaces. Classic snaps run unconfined with full access, like a normally installed program, and must be requested explicitly with `--classic`.

</details>

---

## Challenge: package forensics

Answer from the package database alone.

1. When was the tower's operating system installed? (The install time of the oldest package is a good proxy.)
2. Which kernel versions are installed, and which one is running?
3. `sshd` is misbehaving after someone "tidied up `/etc`". Establish which files belonging to `openssh-server` differ from the packaged versions, and get a pristine copy of `sshd_config` to compare against, **without** overwriting the live one.
4. Produce a list of every package installed or upgraded in the last 30 days, with dates, newest first.
5. A colleague asks which CVEs the installed `openssl` has been patched for. Where do you look?

<details><summary>Solutions</summary>

```bash
# 1
rpm -qa --qf '%{INSTALLTIME} %{INSTALLTIME:date} %{NAME}\n' | sort -n | head -1 | cut -d' ' -f2-

# 2
rpm -q kernel; uname -r

# 3
rpm -V openssh-server
cd ~/labwork/10 && dnf download openssh-server
rpm2cpio openssh-server-*.rpm | cpio -idmv './etc/ssh/sshd_config'      # extract one file into ./etc/ssh/
diff ./etc/ssh/sshd_config /etc/ssh/sshd_config

# 4
rpm -qa --qf '%{INSTALLTIME} %{INSTALLTIME:date} %{NAME}-%{VERSION}-%{RELEASE}\n' \
  | awk -v c="$(date -d '30 days ago' +%s)" '$1 > c' | sort -rn | cut -d' ' -f2-

# 5
rpm -q --changelog openssl | grep -i cve | head -20
```

</details>

## Check yourself

<details><summary>What is the division of labour between <code>rpm</code> and <code>dnf</code>?</summary>
<code>rpm</code> installs, removes, queries and verifies individual packages against the local database. <code>dnf</code> adds repositories, dependency resolution, signature checking of downloads and transaction history, and calls rpm to do the work.
</details>

<details><summary>You need the <code>dig</code> command and <code>dnf install dig</code> finds nothing. What do you run?</summary>
<code>dnf provides dig</code> or <code>dnf provides '*/bin/dig'</code>, which reports <code>bind-utils</code>.
</details>

<details><summary>What does <code>gpgcheck=1</code> protect you from, and what does it not?</summary>
It ensures a package was signed by a key you trusted and has not been altered in transit or on a mirror. It does not protect you from a malicious or compromised publisher: adding a repository is a decision to trust its owner with root.
</details>

<details><summary>Why can three kernels be installed at once when every other package has one version?</summary>
Kernels are "install-only": a new one is added beside the old ones, up to <code>installonly_limit</code>, so that an earlier kernel can be chosen at the boot menu if the new one fails.
</details>

<details><summary>After <code>dnf upgrade</code> replaces openssl, are your running services using the patched library?</summary>
Not until they are restarted. A running process keeps the old file mapped in memory. <code>dnf needs-restarting -s</code> lists the affected services and <code>-r</code> says whether a reboot is needed.
</details>

<details><summary>What does <code>S.5....T. c /etc/chrony.conf</code> from <code>rpm -V</code> mean?</summary>
The size, checksum and modification time differ from what the package installed, and the file is a configuration file. Someone edited it, which is normal for a config file.
</details>

<details><summary>What is the apt equivalent of <code>rpm -qf /path</code>? Of <code>dnf provides</code>?</summary>
<code>dpkg -S /path</code> for installed packages. <code>apt-file search</code> for packages not yet installed.
</details>

## Cleanup

```bash
sudo dnf remove -y zsh mc words 2>/dev/null
sudo dnf versionlock clear 2>/dev/null
rm -rf ~/labwork/10
dnf history | head -8                                       # review what this lab did
```

Keep `dnf-utils` and `tree`. If you installed snapd, follow the removal steps in section 5.
