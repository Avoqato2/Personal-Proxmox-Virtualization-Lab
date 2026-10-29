# LXC 102: Apps (Arcane, Twingate)

Unprivileged LXC container with Docker. It hosts the Arcane manager (Docker management UI), a second Twingate Connector and is the planned home of Nextcloud.

## 1. Container

Specification: 2 cores, 2048 MiB RAM, 2048 MiB swap, 32 GB on `local-lvm`. In contrast to the other containers, the DNS server is set explicitly to the Pi-hole.

Enable `nesting=1` and install Docker from the official repository: https://docs.docker.com/engine/install/debian/

## 2. Arcane

[Arcane](https://github.com/getarcaneapp/arcane) is the central UI for all Docker hosts of the lab.

Agents run on the other Docker hosts (LXC 100 and the Raspberry Pi): add a new environment in the Arcane UI, Arcane then generates the compose file of the agent. Store it on the target host (`~/docker/arcane-agent/docker-compose.yml`) and start it with `docker compose up -d`.

`~/docker/arcane/docker-compose.yml`:

```yaml
services:
  arcane:
    image: ghcr.io/getarcaneapp/manager:latest
    container_name: arcane
    ports:
      - '3552:3552'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - arcane-data:/app/data
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    environment:
      - APP_URL=<app-IP>
      - PUID=1000
      - PGID=1000
      - ENCRYPTION_KEY=<ENCRYPTION_KEY>
    restart: unless-stopped

volumes:
  arcane-data:
```

The web UI is available on port 3552. `ENCRYPTION_KEY` is a secret you generate yourself. Store it safely (e.g. in the password manager), it is needed to read the stored data after a restore.

> [!WARNING]
> Mounting `/var/run/docker.sock` gives Arcane full control over Docker on this host, which is equivalent to root access. Do not expose the UI outside the LAN.

## 3. Twingate Connector (second connector)

A second connector provides redundancy to the one on the Raspberry Pi. It uses the same compose structure as in the [Raspberry Pi documentation](raspberry-pi.md#31-docker-compose), but with its **own** connector tokens.

## 4. Nextcloud (planned)

Nextcloud is planned as personal cloud for phone and web UI.