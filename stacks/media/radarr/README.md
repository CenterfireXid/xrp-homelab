# Radarr

Radarr manages the movie library for XRP Homelab.

It receives indexers from Prowlarr, sends downloads to qBittorrent, and imports completed media into the NAS-backed movie library used by Jellyfin.

> [!WARNING]
> **Legal notice:** This service is documented for educational and personal homelab use only. Use it only to manage, request, index, or download media that you legally own or are otherwise authorized to access. The operator is responsible for complying with applicable copyright laws, licenses, and service terms.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Image | `lscr.io/linuxserver/radarr:6.4.4` |
| Installed version | `6.4.4.10685` |
| Container | `radarr` |
| Port | `7878` |
| Compose path | `/opt/stacks/radarr/compose.yaml` |
| Persistent data | `/srv/docker/radarr` |
| Media mount | `/mnt/unas-media:/data` |

## Architecture

```text
Prowlarr
  ↓
Radarr
  ↓
qBittorrent
  ↓
/data/downloads/complete
  ↓
Radarr import
  ↓
/data/movies
  ↓
Jellyfin
```

Downloads and the movie library are exposed through the same `/data` mount so Radarr can use hardlinks when the underlying filesystem supports them.

## Docker Compose

```yaml
services:
  radarr:
    image: lscr.io/linuxserver/radarr:6.4.4
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=988
      - TZ=America/Denver
    ports:
      - "7878:7878"
    volumes:
      - /srv/docker/radarr:/config
      - /mnt/unas-media:/data
```

`PGID=988` matches the group used for write access to the NFS-backed media share.

## Configuration

### Root Folder

Configure:

```text
/data/movies
```

Do not use the downloads directory as a Radarr root folder.

### qBittorrent

qBittorrent reports completed downloads under:

```text
/downloads
```

Radarr sees the same host storage under:

```text
/data/downloads
```

Configure a Remote Path Mapping in:

```text
Settings → Download Clients → Remote Path Mappings
```

using:

```text
Host: <exact qBittorrent host configured in Radarr>
Remote Path: /downloads
Local Path: /data/downloads
```

The host value must exactly match the qBittorrent host configured in Radarr.

### Prowlarr

Prowlarr is configured under Radarr's indexer integration through Full Sync.

## Initial Deployment

Create directories:

```bash
sudo mkdir -p /opt/stacks/radarr
sudo mkdir -p /srv/docker/radarr
sudo chown -R 1000:1000 /srv/docker/radarr
```

Place `compose.yaml` at:

```text
/opt/stacks/radarr/compose.yaml
```

Validate:

```bash
cd /opt/stacks/radarr
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
docker exec radarr sh -lc '
ls -ld \
  /data \
  /data/downloads \
  /data/downloads/complete \
  /data/movies
'
```

Then configure authentication, `/data/movies`, qBittorrent, the Remote Path Mapping, and Prowlarr.

## Access

```text
http:/<PI_IP>:7878
```

Radarr is intended for private network access and is not exposed directly to the public Internet.

## Verified Import Behavior

The current NFS-backed layout successfully supports hardlink imports.

A completed movie was verified with the same device and inode in both the qBittorrent completed-download path and the Radarr movie-library path:

```text
Same device
Same inode
Link count: 2
```

This confirms Radarr created a hardlink rather than a second full copy.

The result allows qBittorrent to continue seeding from the completed-download path while Jellyfin reads the Radarr-managed library path without duplicating the media data.

## Troubleshooting

### Downloaded - Waiting to Import

**Symptom**

Radarr reports:

```text
No files found are eligible for import in:
/downloads/complete/...
```

while the completed file is visible under:

```text
/data/downloads/complete/...
```

**Cause**

qBittorrent and Radarr use different container paths for the same host storage.

**Fix**

Configure:

```text
Remote Path: /downloads
Local Path: /data/downloads
```

Then refresh the Radarr queue.

### Verify a hardlink

Find the imported file and compare both paths:

```bash
stat -c '%n | dev=%d inode=%i links=%h size=%s' \
'/mnt/unas-media/downloads/complete/<downloaded-file>' \
'/mnt/unas-media/movies/<movie-folder>/<movie-file>'
```

If both entries have the same device and inode, the import is a hardlink.

### Media path is not writable

Confirm the container identity:

```bash
docker exec radarr id
```

Confirm path permissions:

```bash
docker exec radarr ls -ld /data /data/downloads /data/movies
```

The current deployment uses `PGID=988` for write access to the media share.

Do not use `chmod 777` as a generic permissions fix.

## Backup and Restore

Back up:

```text
/srv/docker/radarr
```

Media under `/mnt/unas-media` is separate from Radarr application state and requires its own storage and backup strategy.

For restore:

1. Restore `/srv/docker/radarr`.
2. Restore `/opt/stacks/radarr/compose.yaml`.
3. Confirm ownership.
4. Run `docker compose config`.
5. Pull and deploy.
6. Verify `/data/movies`, qBittorrent, the Remote Path Mapping, and Prowlarr.

## Update Procedure

Check Radarr and LinuxServer release notes before changing the pinned image tag.

```bash
cd /opt/stacks/radarr
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

After updating, confirm access to:

```text
/data/downloads
/data/movies
```

## Portainer

Portainer automatically detects the Compose-created `radarr` container.

Docker Compose remains the deployment source of truth.

## References

- Radarr: https://radarr.video/
- Radarr GitHub: https://github.com/Radarr/Radarr
- LinuxServer Radarr: https://docs.linuxserver.io/images/docker-radarr/
