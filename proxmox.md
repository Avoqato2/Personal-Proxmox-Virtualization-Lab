# Proxmox VE

Core virtualization layer of the lab. It runs all LXC containers, the network bridge and the ZFS pool.

## 1. Hardware

| Component | Specification |
| :--- | :--- |
| CPU | AMD Ryzen 3 1200, 4 cores |
| RAM | 16 GB (ZFS ARC limited to 4 GiB, see [4.4](#44-limit-the-zfs-arc)) |
| Boot SSD | 250 GB: Proxmox OS, `local-lvm` (LVM-Thin) for the container root disks |
| Data HDDs | 2x 2 TB, ZFS mirror (`nas-pool`) |
| Cache SSD | 80 GB, ZFS L2ARC read cache |
| GPU | NVIDIA GTX 1060, installed but currently unused |

Layout of the boot SSD (installer defaults): 8 GB swap, 68 GB root, remaining space as thin pool `data`.

## 2. Installation

Install Proxmox VE with the official ISO and the installer defaults on the boot SSD: https://www.proxmox.com/en/products/proxmox-virtual-environment/get-started

Set the host DNS server to the Raspberry Pi (`Node > System > DNS`, `<Pi-adress>`). Containers without their own `nameserver` inherit this setting.

## 3. Network and Wake-on-LAN

A standard Linux bridge on the physical NIC. The `post-up` line enables Wake-on-LAN (magic packet) on the NIC so UpSnap can start the host.

```bash
nano /etc/network/interfaces
```

```
auto lo
iface lo inet loopback

iface nic0 inet manual
        post-up /sbin/ethtool -s nic0 wol g

auto vmbr0
iface vmbr0 inet static
        address <proxmox-IP>/24
        gateway <gateway-IP>
        bridge-ports nic0
        bridge-stp off
        bridge-fd 0

iface nic1 inet manual

source /etc/network/interfaces.d/*
```

> [!NOTE]
> Interface names (`nic0`, `nic1`) depend on your hardware. Check with `ip link`. Verify Wake-on-LAN with `ethtool nic0 | grep Wake-on` (should show `g`).

## 4. Storage

### 4.1 ZFS pool

Created in the web UI under `Node > Disks > ZFS > Create: ZFS`:

| Setting | Value |
| :--- | :--- |
| Name | `nas-pool` |
| RAID level | Mirror (two 2 TB HDDs) |
| Compression | on |
| ashift | 12 |
| Add Storage | yes |

Select the disks by their stable IDs (`/dev/disk/by-id/...`). Pool properties are otherwise left at their defaults (`recordsize=128K`, `atime=on`, `xattr=sa`).

> [!TIP]
> A pool can also start as a single disk and be extended to a mirror later with `zpool attach nas-pool <existing-disk> <new-disk>`. The resilver runs in the background.

### 4.2 L2ARC cache

The 80 GB SSD is added as read cache:

```bash
zpool add nas-pool cache /dev/disk/by-id/<CACHE-SSD-ID>
```

### 4.3 Datasets

```bash
zfs create nas-pool/data              # NAS data, bind-mounted into LXC 103
zfs create nas-pool/proxmox-backups   # target for vzdump backups
```

### 4.4 Limit the ZFS ARC

By default ZFS may use up to half of the RAM. The ARC is limited to 4 GiB so the containers keep enough memory:

```bash
echo "options zfs zfs_arc_max=4294967296" > /etc/modprobe.d/zfs.conf
update-initramfs -u -k all
```

Apply immediately without reboot and verify:

```bash
echo 4294967296 > /sys/module/zfs/parameters/zfs_arc_max
cat /sys/module/zfs/parameters/zfs_arc_max
```

### 4.5 Storage definitions

`Datacenter > Storage` (`/etc/pve/storage.cfg`):

| ID | Type | Content | Path / Pool |
| :--- | :--- | :--- | :--- |
| `local` | Directory | ISO, templates, import | `/var/lib/vz` |
| `local-lvm` | LVM-Thin | Disk images, containers | thin pool `data` |
| `nas-pool` | ZFS | Disk images, containers | pool `nas-pool` |
| `Backup-VIMS` | Directory | VZDump backup file | `/nas-pool/proxmox-backups` |

`Backup-VIMS` is the backup target.


## 5. Power schedule

The host is shut down every night at 02:00 and started again at 06:00. Both times are scheduled with the cron feature of UpSnap on the Raspberry Pi.
