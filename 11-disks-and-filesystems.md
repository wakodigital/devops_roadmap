# Lab 11 — Disks and Filesystems

**Roadmap topics:** Inodes · Filesystems · Mounts · Adding Disks · Swap · LVM
**Time:** about 4 hours
**Touches system state:** yes, carefully. Every "disk" in this lab is an ordinary **file** attached as a loop device, so no real disk is partitioned, formatted or resized. You will add one line to `/etc/fstab` (backed up first, removed in Cleanup) and create an LVM volume group named `labvg`. Your real disks, real swap and any existing volume group are only looked at.

## The one rule

Commands in this lab that write to a device (`parted`, `mkfs`, `pvcreate`, `mkswap`, `dd of=`) destroy whatever was there. They are safe here because they are aimed at `/dev/loopN` devices that you created. **Before pressing Enter on any of them, read the device name.** If it does not start with `/dev/loop` or `/dev/labvg`, stop.

## What you will be able to do

- Read the storage layout of any Linux machine: disks, partitions, volumes, filesystems, mount points.
- Take a new disk from "plugged in" to "mounted at boot".
- Explain inodes and diagnose a filesystem that is full with space remaining.
- Add swap.
- Build, grow, shrink, snapshot and migrate LVM volumes.

## The storage stack

```
   /mnt/data   /var/lib/docker        mount points: where it appears in the tree
        │            │
   ┌────┴────┐  ┌────┴────┐
   │   xfs   │  │  ext4   │           filesystems: files, directories, inodes
   └────┬────┘  └────┬────┘
   ┌────┴────────────┴────┐
   │  logical volumes     │           LVM (optional): flexible volumes
   │  volume group        │
   │  physical volumes    │
   └────┬────────────┬────┘
   ┌────┴────┐  ┌────┴────┐
   │ sda2    │  │ sdb1    │           partitions: slices of a disk
   └────┬────┘  └────┬────┘
   ┌────┴────┐  ┌────┴────┐
   │  sda    │  │  sdb    │           block devices: disks
   └─────────┘  └─────────┘
```

Each layer is a block device handed to the layer above. A filesystem can sit directly on a partition, or even on a whole disk. LVM is an optional layer in between.

## Setup

```bash
sudo dnf install -y lvm2 parted xfsprogs e2fsprogs util-linux

sudo mkdir -p /var/tmp/labdisks /mnt/lab
cd /var/tmp/labdisks
sudo truncate -s 1G disk1.img disk2.img disk3.img disk4.img     # sparse: they take no real space until written
ls -lh; du -h *.img

sudo cp -a /etc/fstab /etc/fstab.bak.lab11                      # safety copy
```

---

## 0. Read the real layout first

Nothing here changes anything.

```bash
lsblk                                   # the tree: disks, partitions, LVM volumes, mount points
lsblk -f                                # with filesystem type, label, UUID and usage
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,ROTA      # ROTA 1 = spinning disk, 0 = SSD
sudo blkid                              # what is on each block device
df -hT -x tmpfs -x devtmpfs -x overlay  # mounted filesystems, type and usage
findmnt --real                          # the mount tree without virtual filesystems
cat /etc/fstab                          # what is mounted at boot
sudo parted -l 2>/dev/null              # partition tables
sudo pvs; sudo vgs; sudo lvs            # LVM, if present (a default Rocky install uses a VG named "rl")
cat /proc/partitions
```

Device names:

| Name | What |
|---|---|
| `/dev/sda`, `/dev/sdb` | SATA, SAS and USB disks, in detection order |
| `/dev/sda1` | first partition of `sda` |
| `/dev/nvme0n1`, `/dev/nvme0n1p1` | NVMe disk and its first partition |
| `/dev/vda` | virtio disk in a VM |
| `/dev/loop0` | a file presented as a block device |
| `/dev/dm-0`, `/dev/mapper/rl-root`, `/dev/rl/root` | device-mapper devices: LVM volumes, encrypted volumes |
| `/dev/sr0` | optical drive |

`sda` and `sdb` can swap between boots. Refer to filesystems by **UUID** or label, never by `/dev/sdX`, in anything persistent.

```bash
ls -l /dev/disk/by-uuid/ /dev/disk/by-id/ | head -20
```

Write down, in one or two lines, how your tower's root filesystem is stacked (for example: `nvme0n1` → partition 3 → LVM `rl` → LV `root` → xfs → `/`). Exercise 0 below asks for it.

### Exercise 0

1. Describe the path from physical disk to `/` on the tower.
2. Which filesystem type holds `/`? `/boot`? `/boot/efi` if present?
3. Does the tower use LVM? If so, how much unallocated space does its volume group have?
4. Is each `/etc/fstab` entry identified by UUID, by label, or by device path?

<details><summary>How to answer</summary>

```bash
lsblk -s "$(findmnt -no SOURCE /)"      # 1: -s prints the device and everything beneath it
findmnt -no FSTYPE / /boot /boot/efi    # 2: xfs, xfs, vfat is typical
sudo vgs                                # 3: the VFree column
grep -vE '^\s*(#|$)' /etc/fstab         # 4
```

`/dev/mapper/…` paths in fstab are fine: LVM names are stable.

</details>

---

## 1. Adding disks

On a real machine this starts with plugging a disk in and finding its name with `lsblk` (compare before and after) or `sudo dmesg -T | tail`. Here, attach a file:

```bash
cd /var/tmp/labdisks
D1=$(sudo losetup -fP --show disk1.img)      # -f first free loop device, -P scan for partitions, --show print its name
echo "$D1"                                   # e.g. /dev/loop0
losetup -a                                   # all attached loop devices and their backing files
lsblk "$D1"
```

Shell variables do not survive a new login. To get them back later:

```bash
D1=$(losetup -j /var/tmp/labdisks/disk1.img | cut -d: -f1); echo "$D1"
```

### Partition it

A disk needs a partition table. **GPT** is the modern format: disks of any size, up to 128 partitions, a backup copy at the end of the disk. **MBR** (called `msdos` or `dos` by the tools) is the legacy format: 2 TB limit, four primary partitions.

```bash
sudo parted -s "$D1" mklabel gpt                         # CHECK $D1 first. This erases the partition table.
sudo parted -s "$D1" mkpart data1 xfs  1MiB 600MiB       # name, fs-type hint, start, end
sudo parted -s "$D1" mkpart data2 ext4 600MiB 100%
sudo partprobe "$D1"                                     # ask the kernel to re-read the table
sudo parted "$D1" print
lsblk "$D1"                                              # loop0p1 and loop0p2 have appeared
```

Starting at 1 MiB aligns partitions to the disk's physical blocks. `parted -s` is the scriptable form. The interactive alternatives are `sudo fdisk "$D1"` (keys: `g` new GPT, `n` new partition, `p` print, `w` write, `q` quit without saving) and `gdisk`. fdisk changes nothing on disk until you press `w`. parted writes immediately.

The fs-type argument to `mkpart` is only a label in the partition table. It does not create a filesystem.

### Exercises 1

1. Attach `disk4.img` as a loop device, give it a GPT table with three partitions of 300 MiB, 300 MiB and the remainder. Show the result.
2. Delete the second partition (`parted … rm 2`) and show the gap with `parted … print free`.
3. Wipe the partition table with `sudo wipefs -a` and detach the loop device. (disk4 is needed clean later.)

<details><summary>Solutions</summary>

```bash
D4=$(sudo losetup -fP --show /var/tmp/labdisks/disk4.img); echo "$D4"
sudo parted -s "$D4" mklabel gpt mkpart a 1MiB 301MiB mkpart b 301MiB 601MiB mkpart c 601MiB 100%
sudo parted "$D4" print
sudo parted -s "$D4" rm 2
sudo parted "$D4" print free
sudo wipefs -a "$D4"                 # removes filesystem and partition-table signatures
sudo losetup -d "$D4"
```

</details>

---

## 2. Filesystems

A filesystem organises a block device into files and directories. Creating one is "formatting".

```bash
sudo mkfs.xfs  -L labxfs  "${D1}p1"          # CHECK the device. -L sets a label
sudo mkfs.ext4 -L labext4 "${D1}p2"
lsblk -f "$D1"
sudo blkid "${D1}p1" "${D1}p2"               # TYPE, LABEL, UUID, PARTUUID
```

The **UUID** is generated when the filesystem is created and stays with it wherever the disk moves. It changes if you reformat.

| | XFS | ext4 |
|---|---|---|
| Default on | RHEL, Rocky, Alma, Fedora Server | Debian, Ubuntu |
| Grow | yes, online (`xfs_growfs`) | yes, online (`resize2fs`) |
| **Shrink** | **no** | yes, offline |
| Inodes | allocated dynamically | fixed number chosen at mkfs |
| Strengths | large files, parallel I/O, very large filesystems | general purpose, mature repair tools |
| Minimum size | 300 MiB with current tools | a few MiB |
| Inspect | `xfs_info` | `tune2fs -l`, `dumpe2fs -h` |
| Check / repair (unmounted) | `xfs_repair -n` / `xfs_repair` | `e2fsck -fn` / `e2fsck -f` |
| Relabel | `xfs_admin -L` | `e2label` |

Others you will meet: `vfat` (the EFI system partition, USB sticks), `btrfs` (snapshots and checksums built in; default on Fedora Workstation and openSUSE), `tmpfs` (RAM-backed), `overlay` (container image layers), `nfs` and `cifs` (network), and `proc`, `sysfs`, `cgroup2` (kernel interfaces that are not storage at all).

```bash
sudo tune2fs -l "${D1}p2" | grep -Ei 'block count|inode count|block size|mount count|features|reserved block count'
sudo xfs_info "${D1}p1" 2>/dev/null || echo "(xfs_info on Rocky 9 may want it mounted; try again after section 3)"
sudo e2fsck -fn "${D1}p2"                    # -n: check, answer "no" to every repair
sudo xfs_repair -n "${D1}p1"
```

**Never run a repairing fsck on a mounted filesystem.** It corrupts it. Modern journaling filesystems replay their journal at mount and seldom need a manual check; when the kernel logs filesystem errors (`dmesg`), unmount, then repair.

ext4 reserves 5% of blocks for root by default, so that a full disk still leaves root room to log in and clean up. On a large data-only volume that is wasted space: `sudo tune2fs -m 1 DEVICE` lowers it.

### Exercises 2

1. What are the UUIDs of your two new filesystems? Print just the two values.
2. How many inodes does the ext4 filesystem have, and what is its block size?
3. Change the ext4 label to `labdata` and confirm.
4. Reformat `${D1}p2` as ext4 again. Did the UUID change?

<details><summary>Solutions</summary>

```bash
sudo blkid -s UUID -o value "${D1}p1" "${D1}p2"                       # 1
sudo tune2fs -l "${D1}p2" | grep -E 'Inode count|Block size'          # 2
sudo e2label "${D1}p2" labdata && sudo blkid "${D1}p2"                # 3
sudo mkfs.ext4 -q -F -L labext4 "${D1}p2" && sudo blkid "${D1}p2"     # 4: yes, a new UUID
```

</details>

---

## 3. Mounts

Mounting attaches a filesystem to a directory, its **mount point**. Whatever was in that directory before is hidden (not deleted) until you unmount.

```bash
sudo mkdir -p /mnt/lab/xfs /mnt/lab/ext4
sudo mount "${D1}p1" /mnt/lab/xfs
sudo mount "${D1}p2" /mnt/lab/ext4

findmnt /mnt/lab/xfs                         # source, type, options
findmnt -R /mnt/lab                          # everything under a path
df -hT /mnt/lab/xfs /mnt/lab/ext4
mount | grep /mnt/lab                        # the classic view
cat /proc/mounts | grep /mnt/lab             # the kernel's own list
ls -la /mnt/lab/ext4                         # ext4 creates lost+found; xfs does not

sudo chown "$USER": /mnt/lab/xfs /mnt/lab/ext4      # ownership belongs to the filesystem's root directory
sudo restorecon -R /mnt/lab/xfs /mnt/lab/ext4       # a new filesystem has no SELinux labels; apply the policy's defaults
ls -Zd /mnt/lab/xfs
echo "hello" > /mnt/lab/xfs/file.txt
```

### Unmounting

```bash
cd /mnt/lab/xfs
sudo umount /mnt/lab/xfs                     # umount: target is busy
sudo lsof +f -- /mnt/lab/xfs 2>/dev/null     # who is using it (dnf install -y lsof)
sudo fuser -vm /mnt/lab/xfs                  # the same question: here, your own shell, sitting in the directory
cd ~
sudo umount /mnt/lab/xfs                     # works. You can give the mount point or the device.
ls /mnt/lab/xfs                              # empty: file.txt lives on the filesystem, not in this directory
sudo mount "${D1}p1" /mnt/lab/xfs; ls /mnt/lab/xfs
```

`umount -l` (lazy) detaches the mount point now and cleans up when the last user lets go. It is a last resort, because writes may still be in flight.

### Mount options

```bash
sudo mount -o remount,ro /mnt/lab/ext4       # change options without unmounting
touch /mnt/lab/ext4/x                        # Read-only file system
sudo mount -o remount,rw /mnt/lab/ext4

sudo mount -o remount,noexec /mnt/lab/ext4
cp /usr/bin/ls /mnt/lab/ext4/myls
/mnt/lab/ext4/myls                           # Permission denied, although the mode is 755
sudo mount -o remount,exec /mnt/lab/ext4
/mnt/lab/ext4/myls | head -2
```

| Option | Meaning |
|---|---|
| `defaults` | `rw,suid,dev,exec,auto,nouser,async` |
| `ro` / `rw` | read-only / read-write |
| `noexec` | files cannot be executed from here |
| `nosuid` | setuid and setgid bits are ignored |
| `nodev` | device files are not interpreted |
| `noatime` | do not update access times on read: fewer writes |
| `nofail` | **do not fail the boot if the device is absent** |
| `x-systemd.device-timeout=10s` | how long boot waits for the device before giving up |
| `noauto` | do not mount at boot or with `mount -a` |
| `user` | allow ordinary users to mount it |
| `loop` | the source is a file: set up a loop device automatically |
| `_netdev` | needs the network: mount after it is up |

`noexec,nosuid,nodev` on `/tmp`, `/var/tmp`, `/dev/shm` and data volumes is standard hardening.

### /etc/fstab: mounting at boot

Mounts made with `mount` are gone after a reboot. Persistent mounts are listed in `/etc/fstab`, one per line, six fields:

```
UUID=3a1f…  /mnt/lab/xfs  xfs  defaults,nofail  0  0
   1             2         3        4           5  6
```

| Field | Meaning |
|---|---|
| 1 | what: `UUID=…`, `LABEL=…`, a `/dev/mapper/…` path, or `server:/export` |
| 2 | where: the mount point (`none` for swap) |
| 3 | filesystem type |
| 4 | options, comma-separated |
| 5 | `dump` flag. Always 0 |
| 6 | fsck order at boot: 1 for `/`, 2 for other ext filesystems, 0 for none. XFS: 0 |

**A mistake in fstab can stop the machine booting.** If a listed filesystem cannot be mounted and the line lacks `nofail`, systemd drops to an emergency shell on the console and waits for a root password. On a headless server that means fetching a monitor and keyboard. So:

- back up the file first (done in Setup)
- use `nofail` on every line that is not essential to boot
- **test every change with `findmnt --verify` and `mount -a` before you ever reboot**

Add the line. This command builds it from the real UUID:

```bash
U=$(sudo blkid -s UUID -o value "${D1}p1"); echo "$U"
echo "UUID=$U  /mnt/lab/xfs  xfs  defaults,nofail,x-systemd.device-timeout=5s  0 0" | sudo tee -a /etc/fstab
tail -3 /etc/fstab

sudo systemctl daemon-reload                 # systemd turns fstab lines into mount units; tell it the file changed
sudo findmnt --verify                        # parses fstab and reports problems
sudo umount /mnt/lab/xfs
sudo mount /mnt/lab/xfs                      # with an fstab entry, the mount point alone is enough
sudo umount /mnt/lab/xfs
sudo mount -a                                # mount everything in fstab that is not yet mounted: the real test
findmnt /mnt/lab/xfs
systemctl status mnt-lab-xfs.mount | head -5 # the generated unit
```

This particular line would not survive a reboot in working order, because the loop device is not re-created at boot. `nofail` and the 5-second timeout are what make that harmless: boot would log a failed mount and carry on. Cleanup removes the line. For a real disk the same steps give a permanent mount.

### Exercises 3

1. Mount the ext4 partition read-only at `/mnt/lab/ro`. Prove you cannot write, then remount it read-write without unmounting.
2. Make a deliberate mistake safely: append an fstab line with a UUID of all zeros and `nofail`. What do `findmnt --verify` and `mount -a` say? Remove the line again.
3. Create a file directly in `/mnt/lab/hidden-test/` (make the directory), then mount the ext4 partition on top of that directory. Where did your file go? Unmount and look again. Why does this matter for "the disk is full but I cannot find the files"?
4. Which mount options is `/` actually mounted with on the tower? `/boot`? `/dev/shm`?

<details><summary>Solutions</summary>

```bash
# 1
sudo umount /mnt/lab/ext4; sudo mkdir -p /mnt/lab/ro
sudo mount -o ro "${D1}p2" /mnt/lab/ro; touch /mnt/lab/ro/x     # Read-only file system
sudo mount -o remount,rw /mnt/lab/ro; sudo touch /mnt/lab/ro/x && echo writable
sudo umount /mnt/lab/ro; sudo mount "${D1}p2" /mnt/lab/ext4

# 2
echo "UUID=00000000-0000-0000-0000-000000000000 /mnt/lab/bad xfs defaults,nofail 0 0" | sudo tee -a /etc/fstab
sudo findmnt --verify            # reports the unreachable source and the missing target directory
sudo mount -a                    # complains that it cannot find the UUID
sudo sed -i '/00000000-0000-0000-0000-000000000000/d' /etc/fstab
sudo systemctl daemon-reload

# 3
sudo mkdir -p /mnt/lab/hidden-test; sudo touch /mnt/lab/hidden-test/underneath
sudo umount /mnt/lab/ext4; sudo mount "${D1}p2" /mnt/lab/hidden-test
ls /mnt/lab/hidden-test          # "underneath" is hidden beneath the mount, still using space on /
sudo umount /mnt/lab/hidden-test; ls /mnt/lab/hidden-test
sudo mount "${D1}p2" /mnt/lab/ext4
# If a service wrote into a mount-point directory while the disk was not mounted, that data fills the
# parent filesystem and is invisible once the disk is mounted. du cannot see it; df counts it.

# 4
findmnt -no OPTIONS / /boot /dev/shm
```

</details>

---

## 4. Inodes

Lab 04 introduced the inode as the record that *is* a file: type, mode, owner, size, timestamps, link count and pointers to data blocks, with names stored separately in directories.

```bash
stat /mnt/lab/xfs/file.txt                   # Inode: number, Links: count, the three timestamps
ls -i /mnt/lab/xfs/file.txt
df -i /mnt/lab/xfs /mnt/lab/ext4 /           # inodes total, used, free
sudo find / -xdev -inum "$(stat -c %i /etc/hostname)" 2>/dev/null     # find a file by inode number
```

The timestamps in `stat`:

| Field | Updated when |
|---|---|
| Access (`atime`) | the content is read (subject to `relatime` / `noatime`) |
| Modify (`mtime`) | the content changes. This is the time `ls -l` shows |
| Change (`ctime`) | the inode changes: content, permissions, owner, link count. Cannot be set by hand |
| Birth | the file was created (where the filesystem records it) |

### Running out of inodes

ext4 fixes its inode count at creation, by default one inode per 16 KiB of space. Fill them all and no new file can be created, however much space is free. Build a tiny filesystem with very few inodes to see it:

```bash
cd /var/tmp/labdisks
sudo truncate -s 16M tiny.img
sudo mkfs.ext4 -q -F -N 64 tiny.img          # -N: number of inodes. mkfs works on a plain file with -F
sudo mkdir -p /mnt/lab/tiny
sudo mount -o loop tiny.img /mnt/lab/tiny    # "-o loop" attaches a loop device for you
sudo chown "$USER": /mnt/lab/tiny
df -h /mnt/lab/tiny; df -i /mnt/lab/tiny     # about 11M free, 53 inodes free

i=0; while touch /mnt/lab/tiny/f$i 2>/dev/null; do i=$((i + 1)); done; echo "created $i files"
touch /mnt/lab/tiny/one-more                 # No space left on device
df -h /mnt/lab/tiny                          # 1% used
df -i /mnt/lab/tiny                          # 100% used
```

`No space left on device` with space left: always run `df -i` next. The fix is deleting files, not adding space. Finding where they are:

```bash
sudo du --inodes -x /mnt/lab/tiny | sort -n | tail -3
# on a real system:  sudo du --inodes -x / 2>/dev/null | sort -n | tail -20
```

Usual culprits: session files, mail queues, package caches, and container image layers.

```bash
rm /mnt/lab/tiny/f*; df -i /mnt/lab/tiny
sudo umount /mnt/lab/tiny
```

XFS creates inodes as needed (within a ceiling of 25% of the space by default), so this failure is much rarer on your tower's XFS filesystems than on ext4.

### Exercises 4

1. Which filesystem on the tower has the highest inode usage percentage? Which directory under it holds the most inodes?
2. Create a file, note its inode and `ctime`. Run `chmod 600` on it. Which timestamps changed? Then append a line. Which changed now?
3. Rename the file with `mv` inside the same directory. Did the inode number change? What does that tell you about what `mv` does?
4. Copy the file with `cp`. Compare inode numbers.

<details><summary>Solutions</summary>

```bash
# 1
df -i -x tmpfs -x devtmpfs -x overlay | sort -k5 -n | tail -1
sudo du --inodes -x --max-depth=2 / 2>/dev/null | sort -n | tail -5

# 2
cd /mnt/lab/ext4; echo a > t; stat t | grep -E 'Inode|Modify|Change'
chmod 600 t; stat t | grep -E 'Modify|Change'       # only Change (ctime)
echo b >> t; stat t | grep -E 'Modify|Change'       # both Modify and Change

# 3
ls -i t; mv t t2; ls -i t2       # same inode: mv within a filesystem only edits the directory entry

# 4
cp t2 t3; ls -i t2 t3            # different inodes: cp creates a new file
```

</details>

---

## 5. Swap

Swap is disk space the kernel uses to hold memory pages that have not been touched recently, freeing RAM for other work. It lets the machine survive a memory spike that would otherwise invoke the OOM killer, at the price of being very much slower than RAM.

```bash
swapon --show                                # active swap areas: type, size, used, priority
free -h
cat /proc/swaps
grep -i swap /etc/fstab
sysctl vm.swappiness                         # 0–200. How willing the kernel is to swap. RHEL default: 60
vmstat 1 3                                   # si / so columns: pages swapped in / out per second
```

Swap *used* is not a problem by itself; idle pages parked there cost nothing. Ongoing `si`/`so` activity means the machine is short of RAM and is thrashing.

Sizing is a judgement, not a formula. With plenty of RAM, a few GB of swap is a cushion. Swap on a partition or LVM volume and swap in a file perform about the same on modern kernels; a file is easier to resize.

### Add a swap file

```bash
sudo dd if=/dev/zero of=/var/tmp/labdisks/swapfile bs=1M count=256 status=progress
sudo chmod 600 /var/tmp/labdisks/swapfile    # swap can contain any process's memory: root only
sudo mkswap /var/tmp/labdisks/swapfile       # write the swap signature
sudo swapon /var/tmp/labdisks/swapfile       # start using it
swapon --show                                # now listed, with a lower priority than existing swap
free -h
sudo swapoff /var/tmp/labdisks/swapfile      # stop using it (pages in it are moved back to RAM first)
swapon --show
```

Use `dd` (or `fallocate` on ext4) so that the file is fully allocated. A sparse file made with `truncate` is rejected: "swapfile has holes".

To keep swap across reboots you would add an fstab line. **Do not add this one**; the form is:

```
/swapfile                                  none  swap  defaults  0 0
UUID=…  (a swap partition or LV)           none  swap  defaults  0 0
```

On a partition or LV the steps are the same without the `dd`: `mkswap DEVICE`, `swapon DEVICE`.

Tuning:

```bash
# sudo sysctl vm.swappiness=10                                       # until reboot
# echo 'vm.swappiness = 10' | sudo tee /etc/sysctl.d/90-swap.conf    # persistent; apply with: sudo sysctl --system
```

Lower values make the kernel prefer dropping page cache over swapping application memory, which suits database servers. Do not run `swapoff -a` on the tower as an experiment: it forces everything in swap back into RAM at once.

Kubernetes note for later: the kubelet has traditionally required swap to be off. k3s is more relaxed, but you will see "disable swap" in most cluster guides.

### Exercises 5

1. How much swap does the tower have, of what type (partition, LV or file), and how much is in use?
2. Enable the lab swapfile with priority 100 (`swapon -p`). Which area would the kernel use first now? Disable it again.
3. Which processes have the most memory swapped out? (Hint: `VmSwap` in `/proc/*/status`.)

<details><summary>Solutions</summary>

```bash
# 1
swapon --show; lsblk | grep -i swap

# 2
sudo swapon -p 100 /var/tmp/labdisks/swapfile; swapon --show     # highest priority is used first
sudo swapoff /var/tmp/labdisks/swapfile

# 3
sudo grep -H VmSwap /proc/[0-9]*/status 2>/dev/null | sort -k2 -n | tail -5
# then: ps -p PID -o pid,cmd
```

</details>

---

## 6. LVM

With plain partitions, a filesystem is stuck with the size and the disk it was given. **LVM** (Logical Volume Manager) puts a layer of indirection between disks and filesystems:

| Term | What | Analogy |
|---|---|---|
| **PV**, physical volume | a disk or partition handed to LVM | a pile of bricks |
| **VG**, volume group | a pool made of one or more PVs | all your bricks in one heap |
| **LV**, logical volume | a virtual partition carved from a VG | a wall built from the heap |
| **PE**, physical extent | the allocation unit, 4 MiB by default | one brick |

What that buys you: grow a volume while it is mounted; add a disk to the pool and extend onto it; move data off a failing disk while in use; take snapshots.

Commands follow a pattern: `pv…`, `vg…`, `lv…` followed by `create`, `s` (short list), `display` (long), `extend`, `reduce`, `remove`.

**If the tower already has a volume group (check `sudo vgs`), every command below names `labvg` explicitly. Do not run any of them against the real VG.**

### Build it

```bash
cd /var/tmp/labdisks
D2=$(sudo losetup -f --show disk2.img); D3=$(sudo losetup -f --show disk3.img); echo "$D2 $D3"

sudo pvcreate "$D2" "$D3"                    # CHECK the devices. Initialise them as PVs
sudo pvs
sudo vgcreate labvg "$D2" "$D3"              # pool them
sudo vgs labvg                               # VSize about 2 GiB, all of it VFree
sudo lvcreate -n data -L 600M labvg          # -L size, -n name
sudo lvcreate -n logs -L 400M labvg
sudo lvs labvg
lsblk "$D2" "$D3"                            # the LVs appear beneath the loop devices
ls -l /dev/labvg/ /dev/mapper/ | grep lab    # two names for each LV, both symlinks to /dev/dm-N
```

Rocky 9 keeps a list of devices LVM is allowed to use, in `/etc/lvm/devices/system.devices`. `pvcreate` adds to it automatically; `sudo lvmdevices` shows it. Cleanup takes the loop devices back out.

An LV is a block device like any other:

```bash
sudo mkfs.xfs  /dev/labvg/data
sudo mkfs.ext4 -q /dev/labvg/logs
sudo mkdir -p /mnt/lab/data /mnt/lab/logs
sudo mount /dev/labvg/data /mnt/lab/data
sudo mount /dev/labvg/logs /mnt/lab/logs
sudo chown "$USER": /mnt/lab/data /mnt/lab/logs
seq 1 100000 > /mnt/lab/data/numbers.txt; md5sum /mnt/lab/data/numbers.txt | tee ~/lab11.md5
df -hT /mnt/lab/data /mnt/lab/logs
```

### Look at it

```bash
sudo pvs                                     # PVs: which VG, size, free
sudo vgs                                     # VGs: number of PVs and LVs, size, free
sudo lvs                                     # LVs: VG, size
sudo lvs -a -o +devices labvg                # which PV each LV actually sits on
sudo pvdisplay -m "$D2"                      # the extent map of one PV
sudo vgdisplay labvg | grep -E 'PE Size|Total PE|Alloc PE|Free  PE'
```

### Grow a volume, online

```bash
df -h /mnt/lab/data
sudo lvextend -r -L +300M /dev/labvg/data    # -r: also grow the filesystem. +300M relative; -L 900M would be absolute
df -h /mnt/lab/data                          # bigger, while mounted, without interruption
sudo lvextend -r -L +200M /dev/labvg/logs
sudo vgs labvg
```

Without `-r` the LV grows and the filesystem inside does not notice. The second step is then yours: `sudo xfs_growfs /mnt/lab/data` (XFS takes the mount point) or `sudo resize2fs /dev/labvg/logs` (ext4 takes the device). Other size forms: `-l +100%FREE` (all remaining space), `-l +50%VG`.

### Shrink a volume

ext4 can shrink, offline. **XFS cannot shrink at all**; the only way down is to create a smaller volume and copy the data across. That is the main practical argument for starting volumes small and growing them.

```bash
sudo umount /mnt/lab/logs
sudo lvreduce -r -L 400M /dev/labvg/logs     # -r shrinks the FILESYSTEM first, then the LV. Confirm if prompted.
sudo mount /dev/labvg/logs /mnt/lab/logs
df -h /mnt/lab/logs
```

Shrinking an LV **without** `-r` cuts off the end of the filesystem and destroys data. Take a backup before any reduction on a real system.

### Snapshots

A snapshot is a frozen view of an LV at one moment. It is copy-on-write: it takes no space at first, then stores the original contents of each block that changes on the origin. Its size is how much change it can absorb.

```bash
sudo lvcreate -s -n data_snap -L 100M /dev/labvg/data       # -s snapshot of data, 100M of change capacity
sudo lvs labvg                                              # Origin column, and Data%: how full the snapshot is

echo "damage" > /mnt/lab/data/numbers.txt                   # ruin the file on the live volume
md5sum /mnt/lab/data/numbers.txt; cat ~/lab11.md5           # checksums differ

sudo mkdir -p /mnt/lab/snap
sudo mount -o ro,nouuid /dev/labvg/data_snap /mnt/lab/snap  # nouuid: XFS refuses two filesystems with one UUID
                                                            # (if it complains about the log, add ",norecovery")
md5sum /mnt/lab/snap/numbers.txt                            # the snapshot still has the original
sudo lvs labvg                                              # Data% has moved
```

Two ways back. Copy the file out of the snapshot, or roll the entire volume back by merging:

```bash
sudo umount /mnt/lab/snap /mnt/lab/data
sudo lvconvert --merge /dev/labvg/data_snap                 # origin returns to the snapshot's state; the snapshot is consumed
sudo mount /dev/labvg/data /mnt/lab/data
md5sum /mnt/lab/data/numbers.txt; cat ~/lab11.md5           # matches again
sudo lvs labvg                                              # data_snap is gone
```

Typical use: snapshot before a risky upgrade, merge if it goes wrong, `lvremove` the snapshot if it goes right. A snapshot that fills to 100% becomes invalid, and one left lying around slows writes, so they are short-lived tools. A snapshot on the same disks is also **not a backup**.

### Add a disk, and retire one

```bash
D4=$(sudo losetup -f --show /var/tmp/labdisks/disk4.img); echo "$D4"
sudo pvcreate "$D4"
sudo vgextend labvg "$D4"                    # the pool grows; any LV can now extend onto the new disk
sudo vgs labvg; sudo pvs

sudo pvmove "$D2"                            # migrate every extent off D2 onto the other PVs, online
sudo pvs                                     # D2 is now entirely free
sudo vgreduce labvg "$D2"                    # remove it from the pool
sudo pvremove "$D2"                          # wipe the LVM label
md5sum /mnt/lab/data/numbers.txt; cat ~/lab11.md5      # the data never went offline and is intact
```

That sequence is how a failing or undersized disk is replaced under a running system.

### Where this goes next

- **Thin provisioning** (`lvcreate --type thin-pool`, then `-V` for virtual size): volumes that claim more space than physically exists and allocate on write, with cheap snapshots.
- **RAID inside LVM** (`lvcreate --type raid1 -m 1`): mirrored or striped LVs without a separate `mdadm` layer.
- **Stratis** and **VDO** on RHEL-family systems: pooled storage, and deduplication with compression.

### Exercises 6

1. Create a third LV, `scratch`, of 200M with ext4, mounted at `/mnt/lab/scratch`. Grow it to 500M in one command, online.
2. How much free space remains in `labvg`? Give `scratch` all of it.
3. Take a snapshot of `logs`, write 50 MB of data to `/mnt/lab/logs` with `dd`, and watch the snapshot's `Data%`. What happens if you write more than the snapshot can hold? Remove the snapshot.
4. On which PV or PVs does `data` live now? Show it.
5. You need to shrink the XFS volume `data` to 500M. Describe the procedure. (Do not attempt `lvreduce` on it.)
6. If the tower uses LVM: how would you give its root filesystem another 10G, assuming the VG has free space? Write the command; do not run it.

<details><summary>Solutions</summary>

```bash
# 1
sudo lvcreate -n scratch -L 200M labvg && sudo mkfs.ext4 -q /dev/labvg/scratch
sudo mkdir -p /mnt/lab/scratch && sudo mount /dev/labvg/scratch /mnt/lab/scratch
sudo lvextend -r -L 500M /dev/labvg/scratch; df -h /mnt/lab/scratch

# 2
sudo vgs labvg -o vg_free
sudo lvextend -r -l +100%FREE /dev/labvg/scratch

# 3   (free some space first: scratch took it all)
sudo umount /mnt/lab/scratch; sudo lvremove -y /dev/labvg/scratch
sudo lvcreate -s -n logs_snap -L 60M /dev/labvg/logs
dd if=/dev/urandom of=/mnt/lab/logs/blob bs=1M count=50 status=none; sudo lvs labvg
# Past 100% the snapshot is marked invalid and can no longer be mounted or merged. The origin is unaffected.
sudo lvremove -y /dev/labvg/logs_snap; rm /mnt/lab/logs/blob

# 4
sudo lvs -o +devices labvg/data

# 5
# XFS cannot shrink. Create a new 500M LV, mkfs.xfs it, mount it, copy with rsync -aHAX (or xfsdump | xfsrestore),
# stop whatever uses the data, do a final sync, swap the mount points, then lvremove the old LV.

# 6
# sudo lvextend -r -L +10G /dev/mapper/rl-root
```

</details>

---

## Challenge: a new data disk, start to finish

Detach and wipe `disk1.img` so it is blank, then do everything from memory, as if a new disk had been installed for container data:

1. Attach it. Initialise it for LVM (whole disk as a PV, no partition table) in a new VG `labdata`.
2. Create an LV `containers` using 80% of the VG.
3. Format as XFS with the label `containers`.
4. Mount at `/mnt/lab/containers` with `noatime,nodev,nosuid`.
5. Add an fstab entry by path (`/dev/mapper/labdata-containers`) with those options plus `nofail`. Verify, unmount, and remount with `mount -a`.
6. Show the complete stack for that mount point in one `lsblk` command.
7. Grow the filesystem into the remaining 20%, online.

<details><summary>Solution</summary>

```bash
sudo umount /mnt/lab/xfs /mnt/lab/ext4 2>/dev/null
sudo sed -i '\|/mnt/lab/xfs|d' /etc/fstab; sudo systemctl daemon-reload
D1=$(losetup -j /var/tmp/labdisks/disk1.img | cut -d: -f1)
sudo wipefs -a "${D1}p1" "${D1}p2" "$D1"; sudo losetup -d "$D1"

D1=$(sudo losetup -f --show /var/tmp/labdisks/disk1.img)                         # 1
sudo pvcreate "$D1" && sudo vgcreate labdata "$D1"
sudo lvcreate -n containers -l 80%VG labdata                                     # 2
sudo mkfs.xfs -L containers /dev/labdata/containers                              # 3
sudo mkdir -p /mnt/lab/containers                                                # 4
sudo mount -o noatime,nodev,nosuid /dev/labdata/containers /mnt/lab/containers
echo "/dev/mapper/labdata-containers /mnt/lab/containers xfs noatime,nodev,nosuid,nofail,x-systemd.device-timeout=5s 0 0" | sudo tee -a /etc/fstab   # 5
sudo systemctl daemon-reload; sudo findmnt --verify
sudo umount /mnt/lab/containers; sudo mount -a; findmnt /mnt/lab/containers
lsblk -s /dev/labdata/containers                                                 # 6
sudo lvextend -r -l +100%FREE /dev/labdata/containers; df -h /mnt/lab/containers # 7
```

On a real disk the only differences are the device name (`/dev/sdb` in place of the loop device), a `smartctl -H` health check first, and no `x-systemd.device-timeout`. Moving Docker's data there would then be: stop Docker, `rsync -aHAX /var/lib/docker/ /mnt/…/`, set `"data-root"` in `/etc/docker/daemon.json`, start Docker.

</details>

## Check yourself

<details><summary><code>touch</code> fails with "No space left on device", and <code>df -h</code> shows 40% used. What is the next command?</summary>
<code>df -i</code>. The filesystem has run out of inodes.
</details>

<details><summary>Why should fstab refer to filesystems by UUID and not <code>/dev/sdb1</code>?</summary>
Kernel device names are assigned in detection order and can change between boots or when a disk is added. The UUID is stored in the filesystem and follows it.
</details>

<details><summary>What does <code>nofail</code> do, and when must you not use it?</summary>
It lets boot continue if the device is missing or the mount fails. Do not put it on filesystems the system cannot run without, where carrying on would be worse than stopping.
</details>

<details><summary>You ran <code>lvextend -L +10G</code> and <code>df</code> shows no change. Why?</summary>
The logical volume grew, but the filesystem inside it was not resized. Run <code>xfs_growfs MOUNTPOINT</code> or <code>resize2fs DEVICE</code>, or use <code>lvextend -r</code> next time.
</details>

<details><summary>Can you shrink an XFS filesystem?</summary>
No. Create a smaller one and copy the data.
</details>

<details><summary>Is an LVM snapshot a backup?</summary>
No. It lives on the same disks as the origin and depends on it. It protects against a bad change, not against losing the disk. It is useful for taking a consistent backup <em>from</em>.
</details>

<details><summary>After editing <code>/etc/fstab</code>, which two commands must succeed before you reboot?</summary>
<code>findmnt --verify</code> and <code>mount -a</code> (after <code>systemctl daemon-reload</code>).
</details>

<details><summary>In <code>pvs</code>, <code>vgs</code>, <code>lvs</code> order: what is each layer made from?</summary>
A PV is a disk or partition initialised for LVM. A VG is a pool of one or more PVs. An LV is a volume allocated from a VG, on which you put a filesystem.
</details>

## Cleanup

Run every block, in order. The order matters: unmount, then LVM, then loop devices, then files.

```bash
cd ~
# 1. take the lab lines out of fstab, and prove it matches the copy made in Setup
sudo sed -i '\|/mnt/lab|d' /etc/fstab
diff /etc/fstab /etc/fstab.bak.lab11 && echo "fstab is back to its original content"
sudo systemctl daemon-reload
sudo findmnt --verify

# 2. unmount everything under /mnt/lab, deepest first
for m in $(findmnt -rno TARGET | grep '^/mnt/lab' | sort -r); do sudo umount "$m"; done
sudo swapoff /var/tmp/labdisks/swapfile 2>/dev/null

# 3. remove the LVM objects (lab VGs ONLY)
sudo vgremove -y labvg labdata 2>/dev/null
for img in /var/tmp/labdisks/*.img; do
  dev=$(losetup -j "$img" | cut -d: -f1)
  [ -n "$dev" ] || continue
  sudo pvremove -y "$dev" 2>/dev/null
  sudo lvmdevices --deldev "$dev" 2>/dev/null
  sudo losetup -d "$dev"
done

# 4. remove the files and directories
sudo rm -rf /var/tmp/labdisks /mnt/lab
sudo rm -f /etc/fstab.bak.lab11          # only once the diff above reported no differences
rm -f ~/lab11.md5
```

Verify that nothing is left:

```bash
losetup -a | grep labdisks          # expect nothing
sudo vgs 2>/dev/null | grep lab     # expect nothing
findmnt | grep /mnt/lab             # expect nothing
grep lab /etc/fstab                 # expect nothing
swapon --show                       # only your real swap
sudo lvmdevices 2>/dev/null         # no loop devices listed
```
