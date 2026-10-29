# Sonarr

Sonarr manages TV series for the XRP Homelab media stack. It receives indexers from Prowlarr, sends approved downloads to qBittorrent, and imports completed episodes into the NAS-backed TV libraries used by Jellyfin.

> [!WARNING]
> **Legal notice:** This service is documented for educational and personal homelab use only. Use it only to manage, request, index, or download media that you legally own or are otherwise authorized to access. The operator is responsible for complying with applicable copyright laws, licenses, and service terms.

## Purpose

```text
Prowlarr
  ↓
Sonarr
  ↓
qBittorrent
  ↓
/data/downloads/complete
  ↓
Sonarr import
  ↓
/data/tv or /data/kids-tv
  ↓
Jellyfin
```

The deployment uses one `/data` mount for downloads and TV library paths so Sonarr can import completed files without crossing unrelated container filesystems.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Deployment method | Docker Compose |
| Image | `lscr.io/linuxserver/sonarr:4.0.20` |
| Installed version | `4.0.20.3014` |
| Container | `sonarr` |
| Host port | `8989` |
| Compose path | `/opt/stacks/sonarr/compose.yaml` |
| Persistent data | `/srv/docker/sonarr` |
| Media mount | `/mnt/unas-media:/data` |

## Architecture

The NAS media share is mounted on the host at `/mnt/unas-media` and exposed to Sonarr as `/data`. Relevant paths are:

```text
/data/downloads
/data/downloads/complete
/data/tv
/data/kids-tv
```

## Directory Structure

```text
/opt/stacks/sonarr/
└── compose.yaml

/srv/docker/sonarr/

/mnt/unas-media/
├── downloads/
│   └── complete/
├── tv/
└── kids-tv/
```

## Docker Compose

```yaml
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:4.0.20
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=988
      - TZ=America/Denver
    ports:
      - "8989:8989"
    volumes:
      - /srv/docker/sonarr:/config
      - /mnt/unas-media:/data
```

## Storage and Permissions

Sonarr application state is stored under `/srv/docker/sonarr`. Media is provided through `/mnt/unas-media:/data`.

The container runs with:

```text
PUID=1000
PGID=988
```

`PGID=988` matches the host group used for write access to the NFS-backed media share. Avoid broad permission changes such as `chmod 777`.

## Configuration

### Root Folders

Configure:

```text
/data/tv
/data/kids-tv
```

Do not use `/data/downloads` as a Sonarr root folder.

### qBittorrent

qBittorrent reports completed downloads under `/downloads`, while Sonarr sees the same storage under `/data/downloads`. Configure a Remote Path Mapping at:

```text
Settings > Download Clients > Remote Path Mappings
```

Use:

```text
Host: <exact qBittorrent host configured in Sonarr>
Remote Path: /downloads
Local Path: /data/downloads
```

The `Host` value must exactly match the qBittorrent host configured in Sonarr.

### Prowlarr

Prowlarr is configured as the indexer-management application for Sonarr using Full Sync.

## Initial Deployment

Create directories:

```bash
sudo mkdir -p /opt/stacks/sonarr
sudo mkdir -p /srv/docker/sonarr
sudo chown -R 1000:1000 /srv/docker/sonarr
```

Then place `compose.yaml` at:

```text
/opt/stacks/sonarr/compose.yaml
```

Validate:

```bash
cd /opt/stacks/sonarr
docker compose config
```

Pull and deploy:

```bash
docker compose pull
docker compose up -d
```

Verify:

```bash
docker compose ps
docker compose logs --tail=100
```

Confirm storage visibility:

```bash
docker exec sonarr sh -lc '
ls -ld \
  /data \
  /data/downloads \
  /data/downloads/complete \
  /data/tv \
  /data/kids-tv
'
```

Then configure authentication, `/data/tv` and `/data/kids-tv`, qBittorrent, the Remote Path Mapping, and Prowlarr.

## Access

```text
http://<PI-IP>:8989
```

Sonarr is intended for private network access and should not be exposed directly to the public Internet.

## Portainer

Portainer automatically detects the Compose-created `sonarr` container. Use Portainer for status, logs, resource usage, console access, network inspection, restarts, and troubleshooting. Docker Compose remains the deployment source of truth.

## Backup

Back up:

```text
/srv/docker/sonarr
```

Media under `/mnt/unas-media` is separate from Sonarr application state and requires its own backup strategy.

## Restore

1. Restore `/mnt/unas-media`.
2. Recreate `/opt/stacks/sonarr` and `/srv/docker/sonarr`.
3. Restore `/srv/docker/sonarr`.
4. Restore ownership on the application-data directory.
5. Restore `compose.yaml`.
6. Run `docker compose config`.
7. Pull and deploy.
8. Verify root folders, qBittorrent, Remote Path Mapping, and Prowlarr integration.

## Update Procedure

Check release notes before changing the pinned image tag. Then:

```bash
cd /opt/stacks/sonarr
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Verify `/data/downloads`, `/data/tv`, and `/data/kids-tv` after the update.

## Troubleshooting

### Downloaded item cannot be imported

If qBittorrent reports a path under `/downloads/...` but Sonarr sees the file under `/data/downloads/...`, configure:

```text
Host: <exact qBittorrent host configured in Sonarr>
Remote Path: /downloads
Local Path: /data/downloads
```

Then refresh the Sonarr queue.

### Media directory is visible but not writable

```bash
docker exec sonarr id
docker exec sonarr ls -ld /data /data/downloads /data/downloads/complete /data/tv /data/kids-tv
```

The current deployment uses `PGID=988` for NFS-backed media write access. Inspect ownership and NFS behavior before making permission changes.

### Jellyfin does not show an imported episode

First confirm the episode exists under `/mnt/unas-media/tv` or `/mnt/unas-media/kids-tv`. If it exists, refresh the Jellyfin library. If it remains only in `/mnt/unas-media/downloads/complete`, inspect Sonarr Activity → Queue for an import error.

## References

- Sonarr: https://sonarr.tv/
- Sonarr GitHub: https://github.com/Sonarr/Sonarr
- LinuxServer Sonarr image: https://docs.linuxserver.io/images/docker-sonarr/
