## 1. What is fstab?

The /etc/fstab file (filesystem table) is a system configuration file that defines how filesystems, partitions, and storage devices are mounted at boot time. The system reads this file during startup and mounts each entry automatically.

Understanding /etc/fstab is essential when you need to add a new disk, create a swap file, mount a network share, or change mount options for an existing filesystem.

## 2. Anatomy of a line

Every active line has six fields separated by spaces or tabs:

&lt;device&gt; &lt;mount point&gt; &lt;type&gt; &lt;options&gt; &lt;dump&gt; &lt;pass&gt;

A realistic example file:

\# /etc/fstab: static file system information  
UUID=3f2a1c9e-8b7d-4c1e-9a55-1d2e3f4a5b6c / ext4 defaults 0 1  
UUID=A1B2-C3D4 /boot/efi vfat umask=0077 0 1  
UUID=9d8c7b6a-1111-2222-3333-444455556666 /data xfs defaults,nofail 0 2  
192.168.1.10:/export/share /mnt/share nfs defaults,\_netdev 0 0  
/swapfile none swap sw 0 0

Lines beginning with # are comments and are ignored. Blank lines are ignored too.

## 3. The six fields explained

### Field 1: Device (the file system to mount)

This identifies _what_ is being mounted. You can write it in several ways:

| Format        | Example              | Notes                                                        |
| ------------- | -------------------- | ------------------------------------------------------------ |
| UUID          | UUID=3f2a1c9e-...    | Recommended. Stable even if disks are reordered.             |
| LABEL         | LABEL=backup         | Readable, but labels can be duplicated by mistake.           |
| PARTUUID      | PARTUUID=0a1b2c3d-01 | Identifies the partition itself rather than the filesystem.  |
| Device path   | /dev/sdb1            | Simple but fragile: names like sdb can change between boots. |
| Network path  | server:/export/share | NFS. For SMB use //server/share.                             |
| Pseudo device | tmpfs, proc          | Used for virtual filesystems.                                |

### Field 2: Mount point

The directory where the filesystem will appear. It must already exist (create it with mkdir -p). For swap, use none.

### Field 3: Filesystem type

Common values: ext4, xfs, btrfs, vfat, ntfs-3g, nfs, nfs4, cifs, swap, tmpfs. You can use auto to let the kernel detect the type, but explicit is better.

### Field 4: Dump

A legacy flag for the old dump backup utility. 0 means do not back up. In practice this is almost always 0.

### Field 5: Pass (fsck order)

Controls the order in which fsck checks filesystems at boot:

- 1: the root filesystem only
- 2: other filesystems to be checked after root
- 0: never checked (use for swap, network shares, and XFS/Btrfs, which do not use fsck at boot)

## 4. Mount options in depth

### General options

| Option        | Meaning                                         |
| ------------- | ----------------------------------------------- |
| defaults      | Shortcut for rw,suid,dev,exec,auto,nouser,async |
| rw / ro       | Read-write / read-only                          |
| auto / noauto | Mount at boot or mount -a / do not              |
| user / nouser | Allow ordinary users to mount / root only       |
| exec / noexec | Allow / forbid running binaries                 |
| suid / nosuid | Honour / ignore setuid and setgid bits          |
| dev / nodev   | Interpret / ignore device files                 |
| sync / async  | Write immediately / buffer writes               |

### Performance options

| Option   | Meaning                                                                                |
| -------- | -------------------------------------------------------------------------------------- |
| noatime  | Do not update access times on reads. Reduces disk writes.                              |
| relatime | Update access time only if older than modify time (the usual default).                 |
| discard  | Send TRIM commands on delete (SSDs). Many admins prefer a weekly fstrim timer instead. |
| commit=N | (ext4) Flush data every N seconds.                                                     |

### Boot-safety options

| Option                       | Meaning                                                       |
| ---------------------------- | ------------------------------------------------------------- |
| nofail                       | Do not fail the boot if the device is absent.                 |
| \_netdev                     | Marks a network device. Mounted only after the network is up. |
| x-systemd.automount          | Mount lazily on first access.                                 |
| x-systemd.device-timeout=10s | Wait only 10 seconds for the device.                          |
| x-systemd.requires=...       | Declare a dependency on another unit.                         |

### Ownership and permissions (for filesystems without Unix permissions)

VFAT, NTFS and CIFS have no native Linux ownership, so you set it at mount time:

umask=022,uid=1000,gid=1000

## 5. Finding the information you need

Before writing any line, gather the facts.

lsblk -f # tree of disks, filesystems, UUIDs, mount points  
sudo blkid # UUIDs and types of all block devices  
findmnt # what is mounted right now  
df -hT # usage and filesystem types

Example lsblk -f output:

NAME FSTYPE LABEL UUID MOUNTPOINT  
sda  
├─sda1 vfat A1B2-C3D4 /boot/efi  
└─sda2 ext4 3f2a1c9e-8b7d-4c1e-9a55-1d2e3f4a5b6c /  
sdb  
└─sdb1 xfs 9d8c7b6a-1111-2222-3333-444455556666

Here sdb1 has a filesystem but is not mounted, so it is a candidate for a new fstab entry.

## 6. Step-by-step: adding a new disk

Suppose you added a new disk /dev/sdb, and you want it mounted on /data.

**Step 1: Create a partition** (skip if it already has one)

sudo parted /dev/sdb --script mklabel gpt mkpart primary 0% 100%

**Step 2: Create a filesystem** (this erases the partition)

sudo mkfs.ext4 /dev/sdb1  
\# or: sudo mkfs.xfs /dev/sdb1

**Step 3: Create the mount point**

sudo mkdir -p /data

**Step 4: Back up fstab**

sudo cp /etc/fstab /etc/fstab.bak.\$(date +%F)

**Step 5: Get the UUID**

sudo blkid /dev/sdb1

**Step 6: Add the line**

sudo nano /etc/fstab

UUID=9d8c7b6a-1111-2222-3333-444455556666 /data ext4 defaults,nofail 0 2

**Step 7: Validate and test**

sudo findmnt --verify  
sudo systemctl daemon-reload  
sudo mount -a  
df -h /data

If mount -a returns no errors and df shows the disk, the entry is good.

**Step 8: Set ownership if needed**

sudo chown -R youruser:youruser /data

## 7. More examples

### Swap file

sudo fallocate -l 4G /swapfile  
sudo chmod 600 /swapfile  
sudo mkswap /swapfile  
sudo swapon /swapfile

/swapfile none swap sw 0 0

### NFS share

192.168.1.10:/export/share /mnt/share nfs defaults,\_netdev,nofail,x-systemd.automount 0 0

### SMB/CIFS share with credentials file

Create /root/.smbcreds (permissions 600):

username=alice  
password=secret  
domain=WORKGROUP

Then:

//server/share /mnt/smb cifs credentials=/root/.smbcreds,uid=1000,gid=1000,\_netdev,nofail 0 0

### Temporary RAM filesystem

tmpfs /tmp tmpfs defaults,noatime,nosuid,nodev,size=2G 0 0

### Read-only data partition

UUID=1111-2222 /archive ext4 ro,noexec,nosuid,nodev,nofail 0 2

### Hardened mount for an upload directory

UUID=aaaa-bbbb /srv/uploads ext4 defaults,noexec,nosuid,nodev 0 2

### Bind mount (make one directory appear in another place)

/srv/data/projects /home/alice/projects none bind 0 0

## 8. Troubleshooting

| Symptom                                          | Likely cause                                                                | Fix                                                                                        |
| ------------------------------------------------ | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| System drops to emergency mode at boot           | Bad UUID, missing disk, or typo, with no nofail                             | Log in to the emergency shell, remount root read-write (mount -o remount,rw /), edit fstab |
| mount: wrong fs type, bad option, bad superblock | Wrong type or missing helper package (e.g. nfs-common, cifs-utils, ntfs-3g) | Check the type with blkid, install the helper                                              |
| mount point does not exist                       | Directory missing                                                           | mkdir -p the directory                                                                     |
| Network share fails at boot                      | Mounted before the network is up                                            | Add \_netdev and consider x-systemd.automount                                              |
| Changes seem ignored                             | systemd has cached old units                                                | sudo systemctl daemon-reload                                                               |
| Permission denied on a VFAT/NTFS/CIFS mount      | No Unix ownership on these types                                            | Add uid=, gid=, umask= options                                                             |
| Mounted read-only unexpectedly                   | Filesystem errors detected                                                  | Check dmesg, then run fsck on the unmounted device                                         |
| Disk name changed after reboot                   | Using /dev/sdX                                                              | Switch to UUID                                                                             |

### Recovering from emergency mode

1. At the prompt, enter the root password (or press Ctrl+D to retry).
2. Remount the root filesystem read-write:

mount -o remount,rw /

3. Restore the backup or fix the faulty line:

cp /etc/fstab.bak /etc/fstab  
\# or: nano /etc/fstab

4. Test with mount -a, then reboot.

If you cannot reach a shell at all, boot from a live USB, mount the root partition, and edit etc/fstab there.

## 9. Best practices

- **Back up before every edit**, ideally with a dated filename.
- **Prefer UUIDs** over /dev/sdX names.
- **Add nofail** to every non-essential disk so a missing device never blocks boot.
- **Add \_netdev** to all network filesystems (NFS, SMB, iSCSI).
- **Always run findmnt --verify and mount -a** before rebooting.
- **Use comments** to record what each line is and why it exists.
- **Leave the root line alone** unless you are sure of what you are changing.
- **Keep secrets out of fstab.** It is world-readable. Use a credentials file with chmod 600.
- **Consider hardening options** (noexec, nosuid, nodev) on upload, temp and data directories.
- **Use x-systemd.automount** for rarely used or slow network shares to speed up boot.
- **Check your edits with a second pair of eyes** on production servers. One misplaced character can stop a boot.