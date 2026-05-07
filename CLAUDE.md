# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

A self-hosted homelab. Each top-level directory is one service (or service group) deployed via its own `docker-compose.yml`. There is no orchestrator above the per-directory composes — bring services up/down individually from inside the service directory.

## Per-service Makefile targets

Almost every service exposes the same targets. Run from the service directory:

- `make up` — `docker compose up -d`
- `make up-cf` — same, with the `cloudflare` profile (starts the `cloudflared` sidecar; only services that publish a tunnel define this)
- `make up-recreate` — force-recreate containers (most services also pass `--remove-orphans`; `media/`, `cloudflare-ddns/`, `proxmox-cf-tunnel/` do not)
- `make down` — `docker compose down`
- `make destroy` — `down --rmi all --volumes --remove-orphans` (wipes images and volumes — destructive)

`audiobookshelf/`, `cloudflare-tunnel/` (empty), `proxmox-cf-tunnel/`, `cloudflare-ddns/`, `media/` are the exceptions to the standard set — check the local Makefile.

## Configuration model

- Every service reads its config from a sibling `.env` file (gitignored — `*/.env` in `.gitignore`). Compose files reference `${VAR}` or `${VAR:-default}`; defaults exist for paths but secrets (Cloudflare tokens, VPN keys, SMTP creds) have no fallback and must be in `.env`.
- Default `PUID=1000`, `PGID=1000`, `TZ=Asia/Manila` are baked into most services.
- Bind-mounts are the norm; named volumes are not used. State lives in `./<service>-data/` or `./data/` next to the compose file.

## Cross-cutting architecture

**Traefik (`traefik/`)** is the reverse proxy. It requires a pre-created external Docker network:

```bash
docker network create traefik-network
```

Other services are *not* currently on this network — they expose host ports directly. Traefik is provisioned but most routing is via Cloudflare Tunnels rather than Traefik labels. The `traefik-network` is only referenced by `traefik/` and `portainer/` composes today.

**Cloudflare Tunnels** are the dominant ingress pattern. Most services define a `<service>-cf-tunnel` container (image `cloudflare/cloudflared`) under `profiles: [cloudflare]` with `network_mode: host`. They are inactive unless started via `make up-cf`, which activates the profile. Each tunnel needs `<SERVICE>_CF_TUNNEL_TOKEN` in the service's `.env`.

**`cloudflare-ddns/`** keeps `bhome.network` and `*.bhome.network` pointed at the host's public IP via the Cloudflare API.

**`media/` is the only multi-container stack and it routes through a VPN.** All *arr apps (`sonarr`, `radarr`, `prowlarr`), `qbittorrent`, `nzbget`, and `overseerr` use `network_mode: "service:gluetun-media"` and `depends_on: [gluetun-media]`. Consequences:
- Their ports are published on the `gluetun-media` container, not on themselves. Editing port mappings on a downstream service has no effect — change them on `gluetun-media`.
- `gluetun-media` is configured for a custom WireGuard provider (ProtonVPN port forwarding). All `GLUETUN_*` and `QBT_PORT` env vars are required.
- Killing or restarting `gluetun-media` drops every other container in the stack.

**`shared/`** holds the media library (`Movies/`, `Series/`, `downloads/`) consumed by `media/` via relative paths like `./../shared/Movies`. Sonarr/Radarr/qBittorrent/NZBGet all expect this layout. Override per-service via `*_DIR` env vars.

**`watchtower/`** auto-updates all running containers nightly at 01:00 (`WATCHTOWER_SCHEDULE=0 0 1 * * *`) with image cleanup, and emails results via SMTP (`MAIL_*` env vars). When pinning a container's image tag, remember Watchtower will still try to pull updates for that tag.

## Adding a new service

Follow the existing pattern: create `<service>/docker-compose.yml` with PUID/PGID/TZ env vars and bind-mounts under `./<service>-data` or `./data`; copy a Makefile from a similar service; if it needs public ingress, add a `<service>-cf-tunnel` sidecar under `profiles: [cloudflare]` and document the `<SERVICE>_CF_TUNNEL_TOKEN` env var. Do not add it to `traefik-network` unless you also intend to wire up Traefik labels.
