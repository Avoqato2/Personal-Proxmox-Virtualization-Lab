# LXC 101: Caddy Reverse Proxy

This unprivileged LXC container routes local HTTP/HTTPS traffic to the correct services across the Proxmox host. It acts as the central entry point for services accessed via domains and handles automatic TLS certificates using a DNS challenge.

## 1. Container Specification

- **OS Type:** Debian (amd64)
- **Type:** Unprivileged Container
- **Resources:** 1 Core, 512 MiB RAM
- **Storage:** 8 GB on `local-lvm`
- **Boot:** Start at boot enabled

## 2. Caddy Installation (with DuckDNS Plugin)

To issue trusted TLS certificates for internal services without exposing ports to the internet, Caddy uses the DNS challenge. This requires a custom Caddy build containing the DuckDNS module compiled via `xcaddy`.

- **xcaddy Installation Guide:** https://github.com/caddyserver/xcaddy
- **DuckDNS Plugin Guide:** https://github.com/caddy-dns/duckdns

Once `xcaddy` is installed on the system, build the custom Caddy binary:
```bash
xcaddy build --with https://github.com/caddy-dns/duckdns

sudo mv caddy /usr/bin/caddy
```

## 3. Configuration (Caddyfile)

The routing rules are defined in the `Caddyfile` located at `/etc/caddy/Caddyfile`.

After modifying the `Caddyfile`, reload the service seamlessly:
```bash
sudo systemctl reload caddy
```