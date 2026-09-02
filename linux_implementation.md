# Linux Migration Plan

## Context
This stack was originally set up and battle-tested on macOS (OrbStack). The goal is to migrate it to a Linux PC that is always on, connected via ethernet, and acts as the permanent home server. The Mac will no longer run the stack.

The repo is on a private GitHub. The Linux machine will clone it fresh and run the same stack with minor environment changes.

Read CLAUDE.md first — it contains all the gotchas, known issues, and architecture decisions made during the original setup.

---

## What needs to change from the Mac setup

### 1. User IDs
On Mac: `PUID=501`, `PGID=20`
On Linux: first user is typically `PUID=1000`, `PGID=1000`
Run `id -u` and `id -g` on the Linux machine to confirm, then set these in `.env`.

### 2. Media path
On Mac: `MEDIA_DIR=/Users/yaas/Media`
On Linux: change to something like `/home/<user>/Media` or `/opt/media` — wherever you want files to live.

### 3. Container runtime
Mac used OrbStack. Linux uses Docker Engine directly. The scripts detect this automatically — no changes needed in code.

### 4. Plex
On Mac, Plex ran as a native macOS app (not in Docker). On Linux, install Plex Media Server natively (download from plex.tv for Linux/Ubuntu/Debian). The compose file also has Jellyfin as a fully-containerized alternative if preferred — set `MEDIA_SERVER=jellyfin` in `.env`.

### 5. `.env` file
This file is gitignored and will NOT be in the repo. It must be created manually on the Linux machine. Use `.env.example` as the template. Values needed:
- `WIREGUARD_PRIVATE_KEY` — same key from ProtonVPN account (same as Mac)
- `WIREGUARD_ADDRESSES` — same IPv4 address from ProtonVPN WireGuard config (IPv4 only, no IPv6)
- `MEDIA_DIR` — Linux path
- `PUID` / `PGID` — Linux user IDs
- `TIMEZONE` — same as before
- `MEDIA_SERVER` — `plex` or `jellyfin`

---

## What stays exactly the same
- All Docker images and docker-compose.yml
- VPN config (same ProtonVPN account, same WireGuard key)
- `SERVER_COUNTRIES=Sweden` in docker-compose.yml
- configure.sh — fully reusable, run it after stack is up
- All indexers will need to be re-added in Prowlarr (config is not in the repo, it lives in ~/Media/config/)
- Seerr will need to be re-linked to Plex

---

## High-level steps for the agent

1. **Prerequisites**: confirm Docker Engine is installed and running, git is available, user has sudo

2. **Clone the repo** from the private GitHub

3. **Create `.env`** from the template with Linux-specific values (paths, user IDs, same VPN key)

4. **Create media directories** — the configure.sh and doctor.sh expect `~/Media/Downloads`, `~/Media/Movies`, `~/Media/TV Shows`, `~/Media/config`

5. **Run doctor.sh** — preflight check, will catch any missing dependencies or misconfigured values before starting

6. **Start the stack** — `docker compose up -d`

7. **Wait for containers to be healthy** — gluetun takes ~15 seconds to establish VPN, others follow

8. **Run health-check.sh** — confirm VPN is connected and all web UIs are reachable

9. **Run configure.sh** — sets up Radarr, Sonarr, Prowlarr, Bazarr, Seerr automatically. This is idempotent and safe to re-run if it fails partway.

10. **Add indexers manually in Prowlarr** — configure.sh adds some but others need to be added via the UI. Working indexers from Sweden exit: YTS, The Pirate Bay, Knaben, LimeTorrents. Blocked from Sweden: 1337x, TorrentGalaxy.

11. **Install and set up Plex** on Linux natively, sign in with the existing Plex account — it will recognise the library if pointed at the same folder structure

12. **Complete Seerr setup** — link to Plex account, connect Radarr and Sonarr (configure.sh handles the API side but Plex OAuth requires a browser step)

13. **Install Tailscale** on the Linux machine for remote access from Mac and phone

14. **Create the missing download subdirectories** if Radarr/Sonarr warn about them:
    `mkdir -p ~/Media/Downloads/complete/radarr ~/Media/Downloads/complete/sonarr`

---

## Known gotchas to watch for

- qBittorrent shares gluetun's network. If gluetun ever restarts, run `docker compose up -d qbittorrent` to reattach it.
- `WIREGUARD_ADDRESSES` must be IPv4 only — gluetun crashes on IPv6 addresses.
- configure.sh generates a random qBittorrent password on first run and saves it to `~/Media/state/.qb_password`. Don't delete this file.
- Radarr "all indexers unavailable" after first setup is usually a cached backoff — reset via System → Status in Radarr UI.
- YTS has API grab rate limits (429). Don't manually search the same movie multiple times in quick succession.
- Speed limits must be set via qBittorrent API, not by editing the config file directly — qBittorrent overwrites the file on shutdown.
