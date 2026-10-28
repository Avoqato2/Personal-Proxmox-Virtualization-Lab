# Raspberry Pi 

This host functions as the primary DNS/DHCP server for the local network and as an endpoint for a Twingate Connector. Since the system runs on an SD card, write operations must be heavily limited to prevent premature wear.

## 1. Initial OS Setup

The foundation is a headless Debian-based OS, installed directly onto the SD card.

- **OS Flashing:** Use the official Raspberry Pi Imager (https://www.raspberrypi.com/software/).
- **Settings:** Select "Raspberry Pi OS Lite (64-bit)" (Headless/No Desktop).
- **Pre-Configuration (Imager Settings):**
  - Enable SSH (use public-key authentication ).
  - Set the hostname (e.g., `pihole`).
  - Configure the default system user.
  - Set a static IP via your router or define it later in `dhcpcd.conf` / NetworkManager.

## 2. Core Services

Before setting up containers, the core network services are installed directly on the OS to ensure network resolution is always available, even if Docker fails.

- **Pi-hole:** Provides network-wide ad blocking and local DNS records. **Installation Guide here:** https://github.com/pi-hole/pi-hole/#one-step-automated-install
- **Unbound:** Configured as a recursive, caching DNS resolver to ensure privacy and prevent DNS leaks. Pi-hole forwards all upstream queries to the local Unbound instance. **Installation Guide here:** https://unbound.docs.nlnetlabs.nl/en/latest/getting-started/installation.html

### 2.1 Local DNS Records
The following essential local domains are configured in Pi-hole to route traffic to the Proxmox host and LXC containers:

| Domain | IP Address | Target Service |
| :--- | :--- | :--- |
| `proxmox.lan` | `192.168.0.X` | Proxmox VE Host |
| `pihole.lan` | `192.168.0.X` | Pi-hole Web UI |
| `vault.lan` | `192.168.0.X` | Vaultwarden (via LXC 101 Caddy) |

> [!NOTE]
> The domain for the Vault must have a valid and genuine certificate. This can be verified using a reverse DNS lookup. A free option for registering a domain is available at [https://www.duckdns.org](https://www.duckdns.org/)


## 3. Docker Installation
Docker is used to run isolated apps like the Twingate Connector and UpSnap.
- **Docker Guide installation**: Get your installlation information here: https://docs.docker.com/engine/install/debian/

```bash
# Add the current user to the docker group
sudo usermod -aG docker $USER
newgrp docker
```

### 3.1 Running the Containers
The configuration for the Twingate Connector and UpSnap is defined as Infrastructure as Code. Simply use the `docker-compose.yml` located in this directory:

## 4 Optimizations for the SD card
To protect the SD card, logs are kept in RAM and docker logging is restricted.

### 4.1 Log2Ram
Log2Ram caches the `/var/log` directories in RAM and writes them to the SD card only periodically.
- **Installation Guide here:** https://github.com/azlux/log2ram

 
- **Configuration:**
```bash
sudo nano /etc/log2ram.conf
```
  - `SIZE=128M` *(Adjust the size based on RAM usage)*
  - `USE_RSYNC=true` *(Highly recommended: only writes modified files (deltas) back to the SD card)*

### 4.2 Docker-Logs and Journald limitation
By default, Docker writes extensive JSON logs to disk. This is redirected to `journald` to enforce global storage limits.
- **Docker Daemon:**
```bash
sudo nano /etc/docker/daemon.json
```
```json
  {
    "log-driver": "journald"
  }
```
```bash
sudo nano /etc/systemd/journald.conf
```
Set the maximum log size to prevent RAM/SD-Card exhaustion:
- `SystemMaxUse=50M`
### 4.3 Pi-hole Database Optimization
Pi-hole constantly writes DNS query data to a local database. 
- Configuration:
```bash
sudo nano /etc/pihole/pihole-FTL.conf
```
Add or change this parameter:
- `DBINTERVAL=60` *(Writes to the database only once every 60 minutes)*