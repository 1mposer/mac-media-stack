# mac-media-stack — Agent Context

## What this is
A self-hosted media automation stack running on macOS via Docker (OrbStack).
User is in UAE. VPN exit is Sweden (ProtonVPN WireGuard via gluetun).

## Stack components
| Container | Port | Role |
|-----------|------|------|
| gluetun | — | WireGuard VPN gateway (all torrent traffic routes through this) |
| qbittorrent | 8080 | Torrent client (network_mode: service:gluetun) |
| prowlarr | 9696 | Indexer manager (syncs to Radarr/Sonarr) |
| radarr | 7878 | Movie automation |
| sonarr | 8989 | TV automation |
| bazarr | 6767 | Subtitle automation |
| flaresolverr | 8191 | Cloudflare bypass (used by Prowlarr) |
| seerr | 5055 | User-facing request UI (connected to Plex) |

Media server: **Plex** (runs natively on macOS, not in Docker).

## Key files
- `.env` — VPN keys, MEDIA_DIR, TIMEZONE, PUID/PGID. **Gitignored. Contains real WireGuard private key.**
- `docker-compose.yml` — all images are SHA256-pinned (see IMAGE_LOCK.md)
- `scripts/configure.sh` — idempotent setup script, safe to re-run
- `scripts/health-check.sh` — quick stack status check
- `scripts/doctor.sh` — preflight checks before first start

## Paths
- Media root: `~/Media`
- Downloads: `~/Media/Downloads` → mounted as `/downloads` in containers
- Configs: `~/Media/config/<service>/`
- State/credentials: `~/Media/state/`
- qBittorrent password: `~/Media/state/first-run-credentials.txt`
- qBittorrent password file (used by configure.sh): `~/Media/state/.qb_password`

## Common operations
```bash
# Start stack
cd ~/mac-media-stack && docker compose up -d

# Health check
bash scripts/health-check.sh

# Run configuration (idempotent, safe to re-run)
bash scripts/configure.sh

# Restart VPN (after changing SERVER_COUNTRIES in docker-compose.yml)
docker compose up -d gluetun
# Then reattach qBittorrent (it shares gluetun's network):
docker compose up -d qbittorrent

# Check VPN IP
docker exec gluetun sh -lc 'cat /tmp/gluetun/ip'
```

## VPN config
- Provider: ProtonVPN WireGuard
- Private key in `.env` as `WIREGUARD_PRIVATE_KEY`
- Address in `.env` as `WIREGUARD_ADDRESSES` — **IPv4 only** (`10.x.x.x/32`). gluetun does not support IPv6 WireGuard addresses; including one causes a crash loop.
- Country set in `docker-compose.yml` as `SERVER_COUNTRIES=Sweden`
- Port forwarding is enabled via `VPN_PORT_FORWARDING=on`

## qBittorrent networking
qBittorrent uses `network_mode: "service:gluetun"` — it shares gluetun's network namespace. Consequence: **whenever gluetun restarts, qBittorrent loses network and must be recreated**:
```bash
docker compose up -d qbittorrent
```

## Indexers (configured in Prowlarr)
Working from Sweden exit:
- **YTS** — movies, high quality encodes, has API grab rate limits (429 if too many grabs)
- **The Pirate Bay** — general, reliable
- **Knaben** — Norwegian meta-search, VPN-friendly
- **LimeTorrents** — general public tracker

Blocked even from Sweden (block VPN/datacenter IPs):
- 1337x — 403 Forbidden
- TorrentGalaxy — redirects API requests

EZTV (TV) — intermittently failing, not critical.

## Download path issue
Radarr/Sonarr expect subdirectories `/downloads/complete/radarr` and `/downloads/complete/sonarr`.
If they warn about missing directories:
```bash
mkdir -p ~/Media/Downloads/complete/radarr ~/Media/Downloads/complete/sonarr
```

## Seerr API key
Located at `~/Media/config/seerr/settings.json` → `.main.apiKey`.
The `/api/v1/settings/main` endpoint requires a session cookie for auth — reading from the file is more reliable.

## configure.sh behaviour
- Generates a random qBittorrent password on first run, saves it to `~/Media/state/.qb_password`
- Subsequent runs read the saved password (idempotent)
- Handles all "already configured" duplicate cases gracefully
- Cardigann indexers (e.g. TorrentGalaxy) fetch schema from Prowlarr's own API rather than hardcoding payloads
- 307 redirects are treated as success (Seerr redirects with 307 before setup is complete)

## Known gotchas
- `set -euo pipefail`: any `grep` in a pipeline that returns no matches exits with code 1 and kills the script. Always append `|| true` to grep commands in pipelines.
- Radarr "all indexers unavailable" is often a cached backoff state. Reset via System → Status action buttons, then search again.
- YTS 429 rate limit: triggered when Radarr grabs multiple releases at once. Self-resolves after a few hours.
- qBittorrent upload cap: `Session\GlobalUPSpeedLimit=1` in qBittorrent.conf caps upload to 1 KB/s. Raising it improves download speeds (peers prioritize uploaders).
