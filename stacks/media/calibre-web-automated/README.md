# Calibre-Web Automated

Calibre-Web Automated (CWA) is the ebook library and ingest service for XRP Homelab. It provides a browser-based Calibre-Web interface, automated ebook ingest and cleanup, metadata management, format conversion, and optional e-reader delivery.

The active Calibre library is stored on the NAS. CWA configuration and processed-book backups remain on the Raspberry Pi SSD.

> **Legal notice:** This service is documented for personal homelab use. Only ingest, convert, manage, or send ebooks that you legally own or are otherwise authorized to use. Upstream CWA does not support piracy of copyrighted material.

## Purpose

CWA provides a managed ebook workflow without requiring the full Calibre desktop interface.

The current workflow is:

```text
Completed ebook
      |
      | manual copy now
      | future automation may copy completed ebooks here
      v
/mnt/unas-media/downloads/cwa-ingest
      |
      v
Calibre-Web Automated
  - ingest
  - EPUB fixing
  - conversion to EPUB when configured
  - metadata fetch/edit
  - cover and metadata enforcement
      |
      v
/mnt/unas-media/books-cwa
      |
      v
CWA web library
```

Auto-Send to eReader is disabled. Manual e-reader delivery is the preferred method.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Deployment method | Docker Compose |
| Image | `ghcr.io/crocodilestick/calibre-web-automated:v4.0.8` |
| Installed CWA version | `v4.0.8` |
| Installed Calibre version | `9.1` |
| Compose project | `calibre-web-automated` |
| Container | `calibre-web-automated` |
| Web port | `8083/tcp` |
| Local config | `/srv/docker/calibre-web-automated/config` |
| Ingest path | `/mnt/unas-media/downloads/cwa-ingest` |
| Active library | `/mnt/unas-media/books-cwa` |

## Architecture

```text
Browser
  |
  v
Raspberry Pi :8083
  |
  v
Calibre-Web Automated
  |                     \
  |                      \ local application state
  |                       v
  |                  /srv/docker/calibre-web-automated/config
  |
  +---- ingest <---- /mnt/unas-media/downloads/cwa-ingest
  |
  +---- library ---> /mnt/unas-media/books-cwa
                         |
                         v
                    NAS NFS storage
```

The NAS media export is mounted on the host at `/mnt/unas-media`. Docker bind-mounts only the CWA ingest and library directories into the container.

`NETWORK_SHARE_MODE=true` is required because the Calibre library is on NFS. CWA uses this mode to avoid SQLite WAL behavior that can be unsafe on network filesystems, skip recursive ownership changes on the network share, and use polling instead of relying on `inotify`.

## Directory Structure

Deployment definition:

```text
/opt/stacks/calibre-web-automated/
└── compose.yaml
```

Local persistent application state:

```text
/srv/docker/calibre-web-automated/
└── config/
    ├── app.db
    ├── cwa.db
    ├── processed_books/
    └── ...
```

NAS-backed storage:

```text
/mnt/unas-media/
├── downloads/
│   └── cwa-ingest/
└── books-cwa/
    ├── metadata.db
    ├── .calnotes/
    └── <author>/<book>/
```

`/mnt/unas-media/books-cwa` is the active Calibre-managed library.

The older `/mnt/unas-media/books` tree is not mounted into CWA. It contained unmanaged ebook files before the CWA migration, so the new library was created separately to avoid placing a new `metadata.db` on top of the old tree.

## Docker Compose

Local deployment definition:

```text
/opt/stacks/calibre-web-automated/compose.yaml
```

Deployed configuration:

```yaml
services:
  calibre-web-automated:
    image: ghcr.io/crocodilestick/calibre-web-automated:v4.0.8
    container_name: calibre-web-automated
    restart: unless-stopped

    environment:
      PUID: "1000"
      PGID: "988"
      TZ: "America/Denver"
      NETWORK_SHARE_MODE: "true"

    ports:
      - "8083:8083"

    volumes:
      - /srv/docker/calibre-web-automated/config:/config
      - /mnt/unas-media/downloads/cwa-ingest:/cwa-book-ingest
      - /mnt/unas-media/books-cwa:/calibre-library
```

## Ports

| Host Port | Container Port | Protocol | Purpose |
| ---: | ---: | --- | --- |
| `8083` | `8083` | TCP | CWA web interface |

## Persistent Storage

| Host Path | Container Path | Contents |
| --- | --- | --- |
| `/srv/docker/calibre-web-automated/config` | `/config` | CWA settings, users, SQLite application databases, logs, processed-book backups, and application state |
| `/mnt/unas-media/downloads/cwa-ingest` | `/cwa-book-ingest` | Temporary ingest handoff directory |
| `/mnt/unas-media/books-cwa` | `/calibre-library` | Calibre `metadata.db`, ebook files, covers, OPF metadata, and Calibre library state |

Files placed in the ingest directory are temporary. CWA removes successfully processed ingest files after import.

The current NFS permission model uses GID `988` for media-share write access. The container runs as UID `1000`, GID `988`.

Do not manually rename or reorganize files below `/mnt/unas-media/books-cwa`. Calibre owns the library directory structure and tracks it in `metadata.db`.

## Networking

CWA uses the default Compose bridge network:

```text
calibre-web-automated_default
```

TCP `8083` is published on the Raspberry Pi.

The NAS export is mounted by the host rather than by Docker. The current host uses an NFS systemd automount equivalent to:

```fstab
<NAS-IP>:/var/nfs/shared/media /mnt/unas-media nfs defaults,_netdev,nofail,x-systemd.automount 0 0
```

Verify the mount with:

```bash
findmnt -T /mnt/unas-media
```

The deployment does not use host networking, privileged mode, Docker socket access, or host-device mappings.

## Configuration

| Setting | Value | Reason |
| --- | --- | --- |
| `PUID` | `1000` | Runs CWA as the normal homelab service user |
| `PGID` | `988` | Matches group write access on the NFS media share |
| `TZ` | `America/Denver` | Matches the homelab timezone |
| `NETWORK_SHARE_MODE` | `true` | Required for the NFS-backed library |
| Auto conversion | Enabled | Normalizes configured source formats |
| Conversion target | `EPUB` | Preferred managed ebook format |
| Retained conversion format | `EPUB` | Keeps EPUB during conversion processing |
| Kindle EPUB fixer | Enabled | Repairs supported EPUB issues during ingest |
| Backup conversions | Enabled | Preserves originals under `/config/processed_books` |
| Backup imports | Enabled | Preserves originals under `/config/processed_books` |
| Ingest automerge | `new_record` | Current duplicate/import behavior |
| Ingest timeout | `15` minutes | Current processing timeout |
| Metadata fetch on ingest | Enabled | Verified during controlled ebook ingest |
| Auto-Send to eReader | Disabled | No books are automatically sent to an e-reader |

Metadata and cover edits were verified to update both the Calibre metadata and the managed EPUB file.

## Initial Deployment

This procedure assumes Docker Engine and Docker Compose are already installed on the XRP Homelab host.

### 1. Check prerequisites and conflicts

Confirm ARM64:

```bash
uname -m
```

Expected:

```text
aarch64
```

Check the web port:

```bash
sudo ss -ltnp | grep ':8083 ' || echo "Port 8083 is free"
```

Check for an existing deployment:

```bash
docker ps -a \
  --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}' \
  | grep -i 'calibre-web-automated' \
  || echo "No existing CWA container found"

ls -ld \
  /opt/stacks/calibre-web-automated \
  /srv/docker/calibre-web-automated \
  2>/dev/null || true
```

Confirm the NAS mount:

```bash
findmnt -T /mnt/unas-media
id
```

The current deployment uses NFS and requires group `988` write access.

### 2. Prepare the NAS storage

The active library directory must either be empty for a fresh CWA library or contain a valid Calibre library with `metadata.db`.

For a fresh deployment:

```bash
mkdir -p /mnt/unas-media/downloads/cwa-ingest
mkdir -p /mnt/unas-media/books-cwa

chmod 0770 /mnt/unas-media/downloads/cwa-ingest
chmod 0770 /mnt/unas-media/books-cwa
```

Verify the current permission model:

```bash
stat -c '%n uid=%u gid=%g mode=%a' \
  /mnt/unas-media/downloads/cwa-ingest \
  /mnt/unas-media/books-cwa

test -w /mnt/unas-media/downloads/cwa-ingest \
  && echo "cwa-ingest: writable" \
  || echo "cwa-ingest: NOT writable"

test -w /mnt/unas-media/books-cwa \
  && echo "books-cwa: writable" \
  || echo "books-cwa: NOT writable"
```

The current export presents these paths with GID `988`. If the group or write permissions are wrong, fix them on the NAS/export side.

Do not use `chmod 777` or recursive client-side `chown` as a generic NFS permissions fix as this grants every user access tot he file directory, this is very insecure.

### 3. Create local deployment directories

```bash
sudo mkdir -p /opt/stacks/calibre-web-automated
sudo mkdir -p /srv/docker/calibre-web-automated/config

sudo chown 1000:988 /srv/docker/calibre-web-automated/config
sudo chmod 0770 /srv/docker/calibre-web-automated/config
sudo chmod 0755 /opt/stacks/calibre-web-automated
```

### 4. Install the Compose definition

Place the repository's `compose.yaml` at:

```text
/opt/stacks/calibre-web-automated/compose.yaml
```

### 5. Validate

```bash
cd /opt/stacks/calibre-web-automated
docker compose config
```

Confirm:

- image `ghcr.io/crocodilestick/calibre-web-automated:v4.0.8`
- host port `8083`
- `PUID=1000`
- `PGID=988`
- `NETWORK_SHARE_MODE=true`
- the three expected bind mounts
- no accidental secrets

Resolve validation errors before continuing.

### 6. Pull the image

```bash
docker compose pull
```

### 7. Deploy

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=150
```

First startup will take longer because the container installs and verifies its bundled Calibre environment before the CWA web server becomes available.

Wait for the container to report `healthy`:

```bash
docker compose ps
```
If youa re installing this on a toaster, it may take a hot minute, so this is the perfect time for a coffee break.

A completed startup includes log entries showing the Calibre setup completed and the web server started on port `8083`.

### 8. Verify storage access from the container

```bash
docker exec calibre-web-automated id abc

docker exec -u abc calibre-web-automated sh -c '
for path in \
  /calibre-library \
  /calibre-library/metadata.db \
  /cwa-book-ingest \
  /config
do
  if [ -w "$path" ]; then
    echo "$path : writable"
  else
    echo "$path : NOT writable"
  fi
done
'
```

For a new library, `metadata.db` is created automatically after startup.

### 9. Complete first-run setup

Open:

```text
http://<PI-IP>:8083
```

Upstream default credentials for a new installation are:

```text
Username: admin
Password: admin123
```

**Change the default credentials immediately.**

Configure the CWA ingest behavior:

- auto conversion enabled
- target format `EPUB`
- Kindle EPUB fixer enabled
- conversion and import backups enabled
- metadata fetch on ingest enabled
- Auto-Send to eReader disabled

### 10. Verify ingest

Do not download files directly into `cwa-ingest`. Finish the download somewhere else, then copy the completed ebook into the ingest folder. Downloading the files directly to the ingest can cause a premature ingest with a non fully downloaded file.

Example:

```bash
cp "/path/to/completed/book.epub" \
  /mnt/unas-media/downloads/cwa-ingest/
```

Follow the logs:

```bash
cd /opt/stacks/calibre-web-automated
docker compose logs -f
```

Verify that the ingest copy disappears after successful processing and that the book appears below `/mnt/unas-media/books-cwa`.

### 11. Verify restart persistence

```bash
cd /opt/stacks/calibre-web-automated

docker compose restart
sleep 15

docker compose ps
curl -I http://127.0.0.1:8083
ls -lah /mnt/unas-media/books-cwa
```

Confirm the existing library remains available after restart.

### 12. Verify in Portainer

Confirm Portainer shows:

- Compose project `calibre-web-automated`
- container `calibre-web-automated`
- image `ghcr.io/crocodilestick/calibre-web-automated:v4.0.8`
- published port `8083/tcp`
- healthy state
- the expected `/config`, `/cwa-book-ingest`, and `/calibre-library` bind mounts

Portainer visibility has been verified for this deployment.

## Access

Local:

```text
http://<PI-IP>:8083
```

The documented deployment is intended for trusted local/private network access.

Manual Send to eReader is available in CWA.

## Portainer

Docker Compose owns the deployment definition. Portainer reads the resulting Docker state and is used for status, health, logs, resource use, networks, mounts, console access, and routine troubleshooting.

Do not recreate CWA through Portainer's Add Container workflow.

## Backup

Back up both persistent areas:

```text
/srv/docker/calibre-web-automated/config
/mnt/unas-media/books-cwa
```

`/srv/docker/calibre-web-automated/config` contains CWA application databases, account and application settings, logs, and `processed_books` backups. Treat this backup as sensitive.

`/mnt/unas-media/books-cwa` contains the Calibre library, including `metadata.db`, ebooks, covers, OPF metadata, and Calibre notes/state.

For a consistent filesystem backup, stop CWA first:

```bash
cd /opt/stacks/calibre-web-automated
docker compose stop
```

Back up local configuration:

```bash
sudo tar -C /srv/docker \
  -czf /path/to/backup/calibre-web-automated-config-$(date +%Y%m%d-%H%M%S).tar.gz \
  calibre-web-automated
```

Back up the Calibre library:

```bash
tar -C /mnt/unas-media \
  -czf /path/to/backup/calibre-web-automated-library-$(date +%Y%m%d-%H%M%S).tar.gz \
  books-cwa
```

Then restart and verify:

```bash
docker compose start
docker compose ps
```

Store backups outside the Raspberry Pi and outside the active NAS library location.

## Restore

### 1. Restore prerequisites

Install Docker and Docker Compose, restore the NAS mount at `/mnt/unas-media`, and confirm group `988` write access.

Create the local paths:

```bash
sudo mkdir -p /opt/stacks/calibre-web-automated
sudo mkdir -p /srv/docker
```

### 2. Restore local CWA configuration

```bash
sudo tar -C /srv/docker \
  -xzf /path/to/backup/calibre-web-automated-config-YYYYMMDD-HHMMSS.tar.gz

sudo chown -R 1000:988 /srv/docker/calibre-web-automated/config
```

### 3. Restore the Calibre library

The target should be empty before restoring a full library backup.

Restore the library using the normal NFS-authorized account rather than forcing ownership from client-side root:

```bash
tar -C /mnt/unas-media \
  -xzf /path/to/backup/calibre-web-automated-library-YYYYMMDD-HHMMSS.tar.gz
```

Verify:

```bash
stat -c '%n uid=%u gid=%g mode=%a' \
  /mnt/unas-media/books-cwa \
  /mnt/unas-media/books-cwa/metadata.db

test -w /mnt/unas-media/books-cwa \
  && echo "library: writable" \
  || echo "library: NOT writable"
```

### 4. Restore the Compose definition

Place the repository's `compose.yaml` at:

```text
/opt/stacks/calibre-web-automated/compose.yaml
```

### 5. Validate and deploy

```bash
cd /opt/stacks/calibre-web-automated

docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=150
```

### 6. Verify

Confirm:

```bash
curl -I http://127.0.0.1:8083
find /mnt/unas-media/books-cwa -maxdepth 1 -name metadata.db -ls
```

Then verify the web UI, user account, library contents, ingest settings, Auto-Send state, and Portainer visibility.

## Update Procedure

CWA is pinned to an exact image version. Review upstream release notes before changing the tag.

Back up both `/srv/docker/calibre-web-automated/config` and `/mnt/unas-media/books-cwa` before a material upgrade.

Edit the image tag in:

```text
/opt/stacks/calibre-web-automated/compose.yaml
```

Then:

```bash
cd /opt/stacks/calibre-web-automated

docker compose config
docker compose pull
docker compose up -d

docker compose ps
docker compose logs --tail=150
```

A new image may spend additional time in `health: starting` while Calibre is installed or verified inside the recreated container.

After an update, verify:

```bash
docker inspect calibre-web-automated \
  --format 'Image={{.Config.Image}} Status={{.State.Status}} Health={{if .State.Health}}{{.State.Health.Status}}{{else}}none{{end}}'

curl -I http://127.0.0.1:8083

find /mnt/unas-media/books-cwa \
  -type f -iname '*.epub' | wc -l

find /mnt/unas-media/downloads/cwa-ingest \
  -maxdepth 1 -type f -printf '%f\n'
```

Verify the UI, library, metadata editing, ingest behavior, and Auto-Send state before updating the repository copy.

## Security Considerations

CWA authentication is enabled. The upstream default admin password **must** be changed during initial setup.

TCP `8083` is plain HTTP in the documented local deployment. Do not expose it directly to the public Internet without a separate authenticated TLS/reverse-proxy design.

The deployment does not require:

- `privileged: true`
- Docker socket access
- host networking
- host-device mappings

The `/config` backup is sensitive because it contains account and application configuration. Do not commit it to GitHub.

The Compose file contains no live credentials and requires no `.env` file in the current deployment.

## Troubleshooting

### Web UI is unavailable immediately after deployment or update

**Symptom**

The container is running but reports `health: starting`, and TCP `8083` does not respond yet.

**Cause**

CWA installs or verifies its Calibre runtime during container startup. A recreated container can take longer to become ready.

**Resolution**

Check:

```bash
cd /opt/stacks/calibre-web-automated

docker compose ps
docker compose logs --tail=200
```

Wait for the Calibre setup to complete and the CWA web server to start. Do not treat `Up` by itself as proof that the application is ready.

### NFS `chown` returns `Operation not permitted`

**Symptom**

A client-side `sudo chown` against `/mnt/unas-media` fails.

**Cause**

The NAS NFS export applies server-side ownership rules/root squashing. Client-side root is not permitted to arbitrarily change NAS ownership.

**Resolution**

Verify numeric permissions and actual write access:

```bash
stat -c '%n uid=%u gid=%g mode=%a' \
  /mnt/unas-media/downloads/cwa-ingest \
  /mnt/unas-media/books-cwa

test -w /mnt/unas-media/books-cwa \
  && echo "library writable" \
  || echo "library NOT writable"
```

The current deployment uses group `988` for write access. Correct NFS permissions on the UNAS/export side if needed.

Do not use `chmod 777` as a generic workaround.

### `processed_books` is not writable

**Symptom**

The container user can write `/config` but cannot write `/config/processed_books`.

**Cause**

The directory was created with ownership that does not match the CWA runtime user.

**Resolution**

Because `/config` is local SSD storage, correct only that local directory:

```bash
sudo chown -R 1000:988 \
  /srv/docker/calibre-web-automated/config/processed_books
```

Verify:

```bash
docker exec -u abc calibre-web-automated sh -c '
test -w /config/processed_books &&
echo "processed_books: writable"
'
```

### PDF conversion is stuck

**Symptom**

A PDF remains at a low conversion percentage and the conversion starts again after a container restart.

**Cause**

The source PDF remains in the watched ingest directory, so CWA sees it again after startup.

**Resolution**

Stop CWA before moving the specific source file out of ingest:

```bash
cd /opt/stacks/calibre-web-automated
docker compose stop

mkdir -p /mnt/unas-media/downloads/cwa-hold

mv "/mnt/unas-media/downloads/cwa-ingest/<stuck-book>.pdf" \
   /mnt/unas-media/downloads/cwa-hold/

cat /srv/docker/calibre-web-automated/config/cwa_ingest_retry_queue
cat /srv/docker/calibre-web-automated/config/cwa_ingest_status
```

If the retry queue is empty, start CWA again:

```bash
docker compose start
docker compose ps
docker compose logs --tail=100
```

Do not delete the source PDF just to cancel the conversion.

### Ebook does not ingest

**Symptom**

A completed ebook remains in `cwa-ingest`.

**Resolution**

Check permissions and logs:

```bash
docker exec -u abc calibre-web-automated sh -c '
test -w /cwa-book-ingest && echo "ingest writable"
'

cd /opt/stacks/calibre-web-automated
docker compose logs --tail=150
```

CWA uses polling because `NETWORK_SHARE_MODE=true`, so detection can take a few seconds.

Do not download an in-progress file directly into the ingest directory. Copy it there only after the download is complete.

## References

- Calibre-Web Automated: https://github.com/crocodilestick/Calibre-Web-Automated
- CWA releases: https://github.com/crocodilestick/Calibre-Web-Automated/releases
- CWA upstream Docker Compose: https://github.com/crocodilestick/Calibre-Web-Automated/blob/main/docker-compose.yml
- CWA README and network-share guidance: https://github.com/crocodilestick/Calibre-Web-Automated/blob/main/README.md
