# Nextcloud Stack (Cloudflare Tunnel)

A self-hosted Nextcloud stack that is reachable from the internet without
opening a single port on your router, even behind CGNAT (carrier-grade NAT,
common on home internet connections with no public IP). It works by having
a small container (`cloudflared`) open an outbound-only tunnel to
Cloudflare, which then routes your domain's traffic through that tunnel.

![status](https://img.shields.io/badge/status-stable-green)
![license](https://img.shields.io/badge/license-MIT-green)

## Why this works behind CGNAT

```
Browser  ──HTTPS──▶  Cloudflare Edge  ◀──outbound tunnel──  cloudflared
(anywhere)           (DNS + TLS)                            (on your home server)
```

Both your browser and the `cloudflared` container only ever open **outbound**
connections toward Cloudflare. Nothing connects inbound into your home
network, so no port forwarding, no dynamic DNS client, and no public IP are
required. See `architecture-diagram-prompt.txt` for an AI image-generation prompt that
produces a clean diagram of this flow, if you want a visual for your own
notes.

## Stack

| Service | Role |
|---|---|
| `cloudflared` | Keeps an outbound tunnel open to Cloudflare, routes traffic to `nextcloud` |
| `nextcloud` | The Nextcloud app itself (Apache + PHP) |
| `db` | MariaDB, stores users, file metadata, shares |
| `redis` | Cache and file locking, keeps Nextcloud responsive under load |

## Prerequisites

- A domain (or subdomain) added to a Cloudflare account, with Cloudflare as
  the DNS provider. A subdomain of a domain you already manage on
  Cloudflare works fine, you do not need a separate domain.
- Docker and Docker Compose on the machine that will run the stack.
- No router configuration needed. This is the entire point of the tunnel.

## 1. Create the Cloudflare Tunnel

1. Log in to the [Cloudflare dashboard](https://dash.cloudflare.com).
2. Go to **Zero Trust** (sometimes labeled **Networks** or
   **Networking**), then **Tunnels** (on newer dashboards this may be
   under **Networking → Connectors**). Cloudflare renames this section
   from time to time, if you don't see it exactly like this, look for
   "Tunnels" in the sidebar.
3. Click **Create a tunnel**, choose **Cloudflared** as the connector
   type, and give it a name (for example `nextcloud-home`).
4. On the install step, select **Docker** as the environment. Cloudflare
   shows a `docker run` command containing a long token starting with
   `eyJ`. Copy just the token value, everything after `--token`. That
   token is this project's `CLOUDFLARE_TUNNEL_TOKEN`.
5. Still in the tunnel setup, go to **Public Hostname** and add one:
   - Subdomain: whatever you want (for example `cloud`)
   - Domain: pick your domain from the dropdown
   - Service: type `HTTP`, address `nextcloud:80` (the service name and
     port from `docker-compose.yml`, not `localhost`, since `cloudflared`
     reaches `nextcloud` over the internal Docker network)
   - Save. Cloudflare creates the DNS record for you automatically, there
     is nothing to add manually in DNS.
6. The token is tied to this one tunnel. If you delete the tunnel later,
   the token stops working and you'll need a new one.

## 2. Configure the stack

```bash
git clone https://github.com/juliomauro/nextcloud-stack-cloudflare.git
cd nextcloud-stack-cloudflare
cp .env.example .env
```

Edit `.env`:

| Variable | What to put |
|---|---|
| `MYSQL_ROOT_PASSWORD` | A strong random password for the database root user |
| `MYSQL_PASSWORD` | A strong random password for the Nextcloud database user |
| `REDIS_PASSWORD` | A strong random password for Redis |
| `NEXTCLOUD_ADMIN_USER` | The Nextcloud admin username you want |
| `NEXTCLOUD_ADMIN_PASSWORD` | A strong password for that admin account |
| `NEXTCLOUD_DOMAIN` | The full hostname you set up in step 1, for example `cloud.example.com` |
| `CLOUDFLARE_TUNNEL_TOKEN` | The token you copied in step 1 |

Never commit `.env`, it holds real secrets. `.gitignore` already excludes
it.

## 3. Start it

```bash
docker compose up -d
```

Give it a minute for Nextcloud to finish its first-run setup, then open
`https://<NEXTCLOUD_DOMAIN>` (the hostname from step 1) in a browser. Log
in with the admin user and password from `.env`.

Local LAN access is also available on port `4000` (`http://<host-ip>:4000`)
if you want a fallback that does not depend on Cloudflare being reachable.
Remove that port mapping in `docker-compose.yml` if you do not want it
exposed on your LAN.

## Updating

```bash
docker compose pull
docker compose up -d
```

Nextcloud runs its own upgrade steps automatically on first boot after a
version bump. Back up your data first, see below.

## Backups

Three things need backing up:

- The `db_data` volume (MariaDB's data directory)
- The `nextcloud_data` volume (uploaded files)
- Your `.env` file (so you can recreate the stack with the same
  credentials)

```bash
# quick manual backup of both volumes into tarballs
docker run --rm -v nextcloud-stack-cloudflare_db_data:/data -v "$(pwd)":/backup alpine \
  tar czf /backup/db_data.tar.gz -C /data .
docker run --rm -v nextcloud-stack-cloudflare_nextcloud_data:/data -v "$(pwd)":/backup alpine \
  tar czf /backup/nextcloud_data.tar.gz -C /data .
```

(Volume names are prefixed with the Compose project name, which defaults to
the folder name. Run `docker volume ls` to confirm the exact names on your
machine.)

## Troubleshooting

- **Tunnel shows as down in the Cloudflare dashboard**: check
  `docker compose logs cloudflared`. A wrong or expired token is the most
  common cause, generate a fresh one from the tunnel's connector page.
- **502 or "Bad Gateway" through the tunnel**: make sure the Public
  Hostname service address is `nextcloud:80`, not `localhost:80` or an
  IP address. `cloudflared` resolves `nextcloud` through Docker's internal
  network, not the host's.
- **Nextcloud rejects the domain ("Access through untrusted domain")**:
  confirm `NEXTCLOUD_DOMAIN` in `.env` matches exactly the hostname you
  configured in the tunnel, then `docker compose up -d` again so the
  `NEXTCLOUD_TRUSTED_DOMAINS` environment variable picks up the change.

## License

MIT, see [LICENSE](LICENSE).
