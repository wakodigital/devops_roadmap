# Linux Roadmap Lab Workbook

A hands-on course that follows the [roadmap.sh Linux roadmap](https://roadmap.sh/linux) section by section: 16 labs covering all of its topics, plus a capstone project. Written for **Rocky Linux 9** on a headless home server that already runs real workloads.

Every lab is done at the terminal. You read a short explanation, run the commands, and then do exercises whose solutions are collapsed until you open them.

## The labs

| # | Lab | Roadmap section | Time | Changes the system? |
|---|---|---|---|---|
| 01 | [Navigation Basics](01-navigation-basics.md) | Navigation Basics | 1.5 h | no |
| 02 | [Editing Files](02-editing-files.md) | Editing Files | 2 h | installs vim |
| 03 | [Shell and Other Basics](03-shell-basics.md) | Shell and Other Basics | 2 h | your `~/.bashrc`; one temporary sudoers file |
| 04 | [Working with Files](04-working-with-files.md) | Working with Files | 2.5 h | no |
| 05 | [Text Processing](05-text-processing.md) | Text Processing | 4 h | no |
| 06 | [Server Review](06-server-review.md) | Server Review | 2 h | read-only; saves one script |
| 07 | [Process Management](07-process-management.md) | Process Management | 3 h | only processes you start |
| 08 | [User Management](08-user-management.md) | User Management | 2.5 h | `lab*` users and groups, removed at the end |
| 09 | [Service Management (systemd)](09-service-management-systemd.md) | Service Management | 3.5 h | `lab-*` units, removed at the end |
| 10 | [Package Management](10-package-management.md) | Package Management | 2.5 h | installs and removes a few small packages |
| 11 | [Disks and Filesystems](11-disks-and-filesystems.md) | Disks and Filesystems | 4 h | loop devices only; one fstab line, removed |
| 12 | [Booting Linux](12-booting-linux.md) | Booting Linux | 2.5 h | read-only; optional reboot |
| 13 | [Networking](13-networking.md) | Networking | 5 h | temporary route, hosts line, firewall port, SSH key |
| 14 | [Shell Programming](14-shell-programming.md) | Shell Programming | 4 h | no |
| 15 | [Troubleshooting](15-troubleshooting.md) | Troubleshooting | 4 h | a private test network, removed at the end |
| 16 | [Containerization](16-containerization.md) | Containerization | 5 h | `lab-*` containers, labelled, removed at the end |
| 17 | [Capstone](17-capstone.md) | all of them | 6–10 h | yes, by design |

About 50 hours of lab time, plus the capstone. At five hours a week that is ten weeks; at ten, five.

The order follows the roadmap and each lab assumes the ones before it. Three labs are reused later: the script from Lab 06 is scheduled in Lab 09, the sample log from Lab 05 feeds the script in Lab 14, and the namespaces of Lab 15 return in Lab 16.

If you have already been working through terminal topics elsewhere (process and performance monitoring, networking tools, text manipulation), Labs 05, 06, 07, 13 and 15 are where that material lives. Do their exercises first and read the explanations only where an exercise stops you.

## How each lab is laid out

1. **Header**: the roadmap topics covered, the time, and exactly what the lab changes on the machine.
2. **What you will be able to do**: check this list again at the end.
3. **Sections**, one per topic or group of topics: explanation, commands to run, then **exercises**.
4. **Solutions** are inside collapsed blocks. Attempt the exercise before opening one. Typing a wrong answer and seeing why it is wrong teaches more than reading a right one.
5. **Challenge**: a larger task with less guidance.
6. **Check yourself**: questions to answer aloud. If you cannot answer one in two sentences, reread that section.
7. **Cleanup**: returns the machine to how it was.

Type the commands. Do not paste them. The typing is where `ss -tulpn` and `journalctl -u` become automatic.

Keep a notes file as you go (`~/linux-notes.md`): commands that surprised you, your own machine's answers to the review questions, and every mistake that cost you more than five minutes.

## Conventions

| Thing | Convention |
|---|---|
| Working directory | `~/labwork/NN/` for lab NN. Created in each lab's Setup, removed in its Cleanup |
| Anything created on the system | named `lab…` (users `labalice`, units `lab-heartbeat`, containers `lab-web`), so it is easy to find and safe to remove |
| `$USER`, `$IF`, `$GW`, `$PID` | shell variables the lab has set. Run the Setup block again in a new terminal |
| `YOUR_CONTAINER`, `NAME`, `PORT` in capitals | a placeholder: substitute your own |
| A command starting with `#` inside a code block | shown for reference. **Do not run it** on the tower |
| `walter` in example output | stands for your username. `whoami` tells you what to use |
| Expected output | shown after a command where the exact result matters. Small differences in order or spacing are normal |

## Working safely on a machine that matters

The tower runs services you care about and is reached over Wi-Fi with no screen attached. The labs are written around that, and these rules apply throughout.

1. **Read the header of each lab** before starting. It says what will change.
2. **Keep a second SSH session open** whenever a lab touches users, sudo, the firewall, SSH or mounts. If the first session is locked out, the second one fixes it.
3. **Never run a lab command against something not named `lab…`** when it stops, kills, deletes, formats or removes. The labs give you their own targets for all of those.
4. **Do the Cleanup section the same day.** It ends with checks that should print nothing. If one prints something, deal with it then.
5. **Validate before you apply**: `sshd -t`, `visudo -c`, `findmnt --verify`, `systemd-analyze verify`, `nginx -t`, `docker compose config`. Every lab uses the relevant one.
6. **Three things are never done as experiments on the tower**: editing boot loader settings (Lab 12), reconfiguring the real network interface (Lab 13), and restarting the Docker daemon or running a blanket `docker … prune` (Lab 16). Each lab explains them and shows how to practise safely.
7. **`tmux`** (Lab 07) protects long work from a dropped Wi-Fi connection. Use it from Lab 07 onward.

If you manage the tower with Ansible: the labs make manual changes and remove them again, so nothing should drift. Anything you decide to **keep** (the review script, the `sc` function, a firewall rule, a timer) belongs in a playbook, so that the machine can be rebuilt with it.

Having a way to get a console (a monitor and keyboard you can carry over) is worth arranging before Labs 11 to 13, where a mistake outside the lab's own instructions could cost you remote access.

## What to install first

Each lab installs what it needs. To do it once:

```bash
sudo dnf install -y vim-enhanced nano tree tmux psmisc procps-ng sysstat strace lsof jq bc zip unzip zstd rsync acl \
  bind-utils iproute iputils nmap-ncat tcpdump traceroute mtr net-tools ipcalc curl wget \
  lvm2 parted xfsprogs e2fsprogs dnf-utils
```

All of these come from Rocky's own repositories. EPEL is not required anywhere; Lab 10 discusses whether to add it.

## Roadmap coverage

Every node on the roadmap, and where it is taught.

| Roadmap section | Topic | Lab § |
|---|---|---|
| **Navigation Basics** | Basic Commands | 01 §1 |
| | Moving Files / Directories | 01 §3 |
| | Creating & Deleting Files / Dirs | 01 §2 |
| | Directory Hierarchy Overview | 01 §4 |
| **Editing Files** | Vim | 02 §2 |
| | Nano | 02 §1 |
| **Shell and Other Basics** | Command Path | 03 §1 |
| | Environment Variables | 03 §2 |
| | Command Help | 03 §3 |
| | Redirects | 03 §4 |
| | Super User | 03 §5 |
| **Working with Files** | File Permissions | 04 §1 |
| | Archiving and Compressing | 04 §2 |
| | Copying and Renaming | 04 §3 |
| | Soft Links / Hard Links | 04 §4 |
| **Text Processing** | stdout / stdin / stderr | 05 §1 (and 03 §4) |
| | pipe | 05 §1 |
| | tee | 05 §1 |
| | head | 05 §2 |
| | tail | 05 §2 |
| | wc | 05 §2 |
| | nl | 05 §2 |
| | cut | 05 §3 |
| | paste | 05 §3 |
| | sort | 05 §4 |
| | uniq | 05 §4 |
| | tr | 05 §5 |
| | expand | 05 §5 |
| | unexpand | 05 §5 |
| | split | 05 §6 |
| | join | 05 §6 |
| | grep | 05 §7 |
| | awk | 05 §8 |
| **Server Review** | Uptime and Load | 06 §1 |
| | Authentication Logs | 06 §2 |
| | Services Running | 06 §3 |
| | Available Memory / Disk | 06 §4 |
| **Process Management** | Background / Foreground Processes | 07 §2 |
| | Listing / Finding Processes | 07 §3 |
| | Process Signals | 07 §4 |
| | Killing Processes | 07 §5 |
| | Process Priorities | 07 §6 |
| | Process Forking | 07 §7 |
| **User Management** | Create / Delete / Update | 08 §2 |
| | Users and Groups | 08 §1, §3 |
| | Managing Permissions | 08 §4 |
| **Service Management (systemd)** | Checking Service Status | 09 §1 |
| | Starting / Stopping Services | 09 §2 |
| | Checking Service Logs | 09 §3 |
| | Creating New Services | 09 §4 (timers §5, troubleshooting §6) |
| **Package Management** | Package Repositories | 10 §1 |
| | Finding & Installing Packages | 10 §2 |
| | Listing Installed Packages | 10 §3 |
| | Install / Remove / Upgrade Packages | 10 §4 |
| | Snap | 10 §5 |
| **Disks and Filesystems** | Adding Disks | 11 §1 |
| | Filesystems | 11 §2 |
| | Mounts | 11 §3 |
| | Inodes | 11 §4 (and 04 §4) |
| | Swap | 11 §5 |
| | LVM | 11 §6 |
| **Booting Linux** | Boot Loaders | 12 §1, §2 |
| | Logs | 12 §3, §4 |
| **Networking** | TCP/IP Stack | 13 §1 |
| | Subnetting | 13 §2 |
| | Ethernet & arp/rarp | 13 §3 |
| | DHCP | 13 §4 |
| | IP Routing | 13 §5 |
| | DNS Resolution | 13 §6 |
| | Netfilter | 13 §7 |
| | SSH | 13 §8 |
| | File Transfer | 13 §9 |
| **Shell Programming** | Literals | 14 §1 |
| | Variables | 14 §2 |
| | Conditionals | 14 §3 |
| | Loops | 14 §4 |
| | Debugging | 14 §6 (functions §5, a complete tool §7) |
| **Troubleshooting** | ICMP | 15 §2 |
| | ping | 15 §3 |
| | traceroute | 15 §4 |
| | netstat | 15 §5 |
| | Packet Analysis | 15 §6 (method §1, fault-finding §7) |
| **Containerization** | ulimits | 16 §2 |
| | cgroups | 16 §3 |
| | Container Runtime | 16 §4 (what a container is §1) |
| | Docker | 16 §5 |

A few things are taught that the roadmap does not list, because the listed topics cannot be used without them: `sed` (05 §9), `find` and `xargs` (throughout), shell functions (14 §5), systemd timers (09 §5), SELinux essentials wherever Rocky's behaviour depends on them (04, 09, 11, 16), and namespaces (16 §1).

## Progress

Tick a box when the lab's "Check yourself" questions are easy and its Cleanup checks came back empty.

- [ ] 01 Navigation Basics
- [ ] 02 Editing Files
- [ ] 03 Shell and Other Basics
- [ ] 04 Working with Files
- [ ] 05 Text Processing
- [ ] 06 Server Review
- [ ] 07 Process Management
- [ ] 08 User Management
- [ ] 09 Service Management (systemd)
- [ ] 10 Package Management
- [ ] 11 Disks and Filesystems
- [ ] 12 Booting Linux
- [ ] 13 Networking
- [ ] 14 Shell Programming
- [ ] 15 Troubleshooting
- [ ] 16 Containerization
- [ ] 17 Capstone

## Where Rocky differs from other distributions

The roadmap is distribution-neutral. These are the places where what you learn here has a different spelling on Debian and Ubuntu.

| Area | Rocky / RHEL family | Debian / Ubuntu |
|---|---|---|
| Packages | `dnf`, `rpm`, `/etc/yum.repos.d/` | `apt`, `dpkg`, `/etc/apt/sources.list.d/` |
| Admin group | `wheel` | `sudo` |
| Auth log | `/var/log/secure` | `/var/log/auth.log` |
| General log | `/var/log/messages` | `/var/log/syslog` |
| Firewall front end | firewalld (`firewall-cmd`) | ufw |
| Mandatory access control | SELinux, enforcing | AppArmor |
| Default filesystem | XFS | ext4 |
| SSH service name | `sshd` | `ssh` |
| Network configuration | NetworkManager (`nmcli`) | netplan, over systemd-networkd or NetworkManager |
| DNS stub | none by default | systemd-resolved at `127.0.0.53` |
| `useradd` creates a home directory | yes | no (use `-m`, or `adduser`) |
| Default container tool | Podman | Docker or Podman, by choice |
| Snap | optional, from EPEL | installed by default |
| `/bin/sh` | bash | dash |

Lab 10 has the full `dnf`-to-`apt` command table.

## How this workbook was checked

Run in a Linux sandbox, with output compared against what the text says:

- the commands in Labs 01, 03, 04, 05 and 14 that need neither root nor Rocky-specific tools, including every expected-output block in Lab 05
- the signal, zombie, fork-tracing and runaway-job demonstrations in Lab 07
- the inode-exhaustion demonstration in Lab 11 and the ulimit and namespace demonstrations in Lab 16
- the `logsum` script and the debugging examples in Lab 14, and the sample web application in Lab 16

Not run before you received it:

- everything Rocky-specific: dnf, systemd units, firewalld, SELinux, LVM, XFS, GRUB, NetworkManager, and Docker on the host. These were written against Rocky 9's documented behaviour.
- the test-network script in Lab 15. Its syntax and its web-server half were checked; its `ip netns` and firewalld commands could not be run in the sandbox.

Where a command behaves differently from the text, trust the machine, read the man page, and note the difference. Differences between Rocky 9 minor releases are the usual reason.

## After the roadmap

The Linux roadmap is the foundation under the DevOps roadmap's later boxes. With these labs done, the natural continuations are: the configuration management and infrastructure-as-code sections (turn each lab's manual steps into a playbook), containers and orchestration (Lab 16 leads straight into Kubernetes), and monitoring (the capstone is a small monitoring system; Prometheus and Grafana are the grown-up versions).
