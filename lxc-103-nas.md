# LXC 103: Samba NAS

Unprivileged LXC container that shares the ZFS dataset `nas-pool/data` through Samba (SMB). It also pulls the backup of the Raspberry Pi.

> [!WARNING]
> Work in progress. The NAS is not fully configured yet; open points are listed at the end.

## 1. Container

Specification: 1 core, 512 MiB RAM, 8 GB on `local-lvm`.

Time zone: set manually inside the container so logs and cron jobs use local time:

```bash
dpkg-reconfigure tzdata      # <TIMEZONE>
```

## 2. ZFS bind mount

The dataset is passed from the Proxmox host into the container as mount point `mp0` (in `/etc/pve/lxc/103.conf`):

```
mp0: /nas-pool/data,mp=/mnt/nas
```

Or from the host shell:

```bash
pct set 103 -mp0 /nas-pool/data,mp=/mnt/nas
```

### File permissions in unprivileged containers

In an unprivileged container, UID/GID `N` inside is `100000 + N` on the host. Root of the container (`0`) is therefore `100000` on the host:

```bash
# on the Proxmox host
chown -R 100000:100000 /nas-pool/data

# inside the container: give the data to the NAS user
chown <NAS_USER>:<NAS_USER> /mnt/nas
```

## 3. Samba

Install Samba from the Debian repositories (`apt install samba`) and create a dedicated user with its own Samba password (`smbpasswd -a <NAS_USER>`). The default `[homes]` share stays disabled. Only the share section is added to `/etc/samba/smb.conf`:

```ini
[NAS]
   path = /mnt/nas
   valid users = <NAS_USER>
   read only = no
```

Check the configuration with `testparm`. Windows clients connect with `\\<nas-IP>\NAS` (or `\\nas.lan\NAS`). Remote access is only possible through the zero-trust tunnel (SMB, port 445).

## 4. Raspberry Pi backup (pull)

This container pulls the backup of the Raspberry Pi, so a compromised Pi can not write into the NAS.

| Item | Value |
| :--- | :--- |
| Schedule | `0 12 * * 1` (every Monday 12:00), root crontab |
| Script | `/root/pi-backup.sh` |
| Transport | `rsync` over SSH with an ed25519 key (no password) |
| Target | `/mnt/nas/backups/raspberry-pi` |
| Sources on the Pi | `~/docker`, `/etc/pihole`, `/etc/unbound` |

The system directories need `sudo rsync` on the Pi, see [Raspberry Pi, section 5](raspberry-pi.md#5-backup-of-the-pi).

`/root/pi-backup.sh`:

```bash
#!/bin/bash
PI_USER="<PI_USER>"
PI_IP="<pi-IP>"
DEST="/mnt/nas/backups/raspberry-pi"

# Creates "backups" and "raspberry-pi" if they do not exist yet
mkdir -p $DEST

# 1. Docker folder (Twingate, UpSnap, Arcane agent)
rsync -avz --delete ${PI_USER}@${PI_IP}:~/docker $DEST/

# 2. Pi-hole and Unbound configuration (needs sudo on the Pi)
rsync -avz --delete ${PI_USER}@${PI_IP}:/etc/pihole $DEST/
rsync -avz --delete ${PI_USER}@${PI_IP}:/etc/unbound $DEST/
```

The script only mirrors the latest state (`--delete`), it does not keep older versions.


## 5. Open points

- Finish share and user concept (further users and shares).
- Remove the unused default `[printers]` and `[print$]` sections from `smb.conf`.
- Data in `/mnt/nas` is a bind mount and therefore **not** part of the Proxmox container backup.