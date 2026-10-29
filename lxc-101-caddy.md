# LXC 101: Caddy Reverse Proxy

Unprivileged LXC container. Caddy terminates HTTPS for services with a domain name and forwards the traffic to the right backend. Certificates are obtained automatically with a DNS challenge, so no port has to be opened to the internet.

## 1. Container

Specification: 1 core, 512 MiB RAM, 8 GB on `local-lvm`.

## 2. DuckDNS

1. Register a free domain at https://www.duckdns.org and note the token (`<DUCKDNS_DOMAIN>`, `<DUCKDNS_TOKEN>`).
2. The domain only has to exist for the DNS challenge. Locally, Pi-hole resolves it to `<vault-IP>`.

## 3. Caddy with DuckDNS plugin

1. Install Caddy from the official Debian repository (https://caddyserver.com/docs/install#debian-ubuntu-raspbian). This provides the `caddy` user, the systemd unit and `/etc/caddy/`. The unit file is unchanged.
2. Install `xcaddy` (https://github.com/caddyserver/xcaddy) and build Caddy with the DuckDNS DNS module (https://github.com/caddy-dns/duckdns):

```bash
xcaddy build --with github.com/caddy-dns/duckdns
sudo systemctl stop caddy
sudo mv caddy /usr/bin/caddy
sudo systemctl start caddy
```

Check that the module is present: `caddy list-modules | grep duckdns`.

> [!NOTE]
> A package update (`apt upgrade`) replaces the custom binary with the stock one. Prevent that with `sudo apt-mark hold caddy` and rebuild manually when you want to update.

## 4. Caddyfile

`/etc/caddy/Caddyfile`:

```
<DUCKDNS_DOMAIN> {
    tls {
        dns duckdns <DUCKDNS_TOKEN>
    }
    reverse_proxy <vault-IP>:8080
}
```

Add one block per service if you route more domains. Apply changes without downtime:

```bash
sudo systemctl reload caddy
```

> [!TIP]
> To keep the token out of the file, use `dns duckdns {env.DUCKDNS_TOKEN}` and set the variable in a systemd override (`sudo systemctl edit caddy`, `[Service]` / `Environment=DUCKDNS_TOKEN=...`).

## 5. Traffic flow

`Client > Pi-hole resolves <DUCKDNS_DOMAIN> to <proxy-IP> > Caddy (HTTPS, port 443) > Vaultwarden (<vault-IP>:8080)`