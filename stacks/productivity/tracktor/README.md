# Tracktor

Tracktor is a self-hosted vehicle management application used on XRP Homelab to keep vehicle records, fuel history, maintenance, documents, reminders, and related information in one place.

## Purpose

I added Tracktor as a local place to maintain vehicle information without depending on a hosted service.

It keeps vehicle history and records in one self-hosted application while leaving the underlying data on the XRP Homelab server.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Deployment method | Docker Compose |
| Image | `ghcr.io/javedh-dev/tracktor:2.0.2` |
| Compose project | `tracktor` |
| Container | `tracktor-app` |
| Restart policy | `unless-stopped` |

Tracktor is pinned to version `2.0.2` (latest at the time of odeployment) rather than `latest` so upgrades can be reviewed and performed deliberately as Tracktor is still listed as under active development and not stable for production.

## Architecture

Tracktor runs as a single Docker container with a bind mount for persistent application data.

```text
Browser
  ↓
Raspberry Pi TCP/3333
  ↓
tracktor-app TCP/3000
  ↓
/srv/docker/tracktor
```

Tracktor uses SQLite for its database. In Docker, the database is stored at `/data/tracktor.db`, while uploaded files are stored under `/data/uploads`.

## Directory Structure

Compose definition:

```text
/opt/stacks/tracktor/
└── compose.yaml
```

Persistent data:

```text
/srv/docker/tracktor/
```

The host directory is mounted to `/data` inside the container.

## Docker Compose

The live deployment is defined at:

```text
/opt/stacks/tracktor/compose.yaml
```

Sanitized configuration:

```yaml
services:
  tracktor:
    image: ghcr.io/javedh-dev/tracktor:2.0.2
    container_name: tracktor-app
    restart: unless-stopped

    ports:
      - "3333:3000"

    volumes:
      - /srv/docker/tracktor:/data

    environment:
      TRACKTOR_DEMO_MODE: "false"
      FORCE_DATA_SEED: "false"
      TRACKTOR_DISABLE_AUTH: "false"
      HTTP_MODE: "http"
```

No credentials or other secrets are stored in this Compose file.

## Ports

| Host Port | Container Port | Purpose |
| ---: | ---: | --- |
| `3333` | `3000` | Tracktor web interface |

## Persistent Storage

| Host Path | Container Path | Contents |
| --- | --- | --- |
| `/srv/docker/tracktor` | `/data` | SQLite database and uploaded files |

The bind mount gives Tracktor's persistent data an explicit host path, which makes backup and recovery easier to understand than relying on an unnamed location inside the container.

## Networking

Tracktor uses the default Docker Compose bridge network.

TCP port `3333` is published on the Raspberry Pi and forwarded to port `3000` inside the container.

No reverse proxy or TLS termination is currently part of this deployment.

## Configuration

| Setting | Value | Reason |
| --- | --- | --- |
| `TRACKTOR_DEMO_MODE` | `false` | Runs the normal application instead of demo mode. |
| `FORCE_DATA_SEED` | `false` | Prevents demo data from being forcibly seeded. |
| `TRACKTOR_DISABLE_AUTH` | `false` | Keeps Tracktor authentication enabled. |
| `HTTP_MODE` | `http` | Matches the current direct HTTP deployment without a TLS-terminating reverse proxy. |

If notification providers are configured later, Tracktor requires an `APP_SECRET` to encrypt stored provider credentials.

## Access

Local access:

```text
http://<PI-IP>:3333
```

## Portainer

Tracktor is defined and deployed through Docker Compose.

Portainer automatically detects the `tracktor` Compose project and `tracktor-app` container and is used for operational visibility, including:

- container status
- logs
- resource usage
- networks
- volumes

Portainer is not the deployment source of truth.

## Backup

The persistent backup target is:

```text
/srv/docker/tracktor
```

This directory contains the Tracktor SQLite database and uploaded files.

For a simple filesystem backup, the safest approach is to stop Tracktor first:

```bash
cd /opt/stacks/tracktor
docker compose stop
```

Back up:

```text
/srv/docker/tracktor
```

Then restart the application:

```bash
docker compose start
```

The Compose definition is stored separately in GitHub.

Tracktor is under active development, so backups should be taken before upgrades.

## Restore

On a clean XRP Homelab host:

1. Install Docker and Docker Compose.

2. Recreate the standard directories:

   ```bash
   sudo mkdir -p /opt/stacks/tracktor
   sudo mkdir -p /srv/docker/tracktor
   ```

3. Restore the backed-up Tracktor data into:

   ```text
   /srv/docker/tracktor
   ```

4. Restore `compose.yaml` to:

   ```text
   /opt/stacks/tracktor/compose.yaml
   ```

5. Set ownership appropriate for the restored files and deployment.

6. Validate the Compose configuration:

   ```bash
   cd /opt/stacks/tracktor
   docker compose config
   ```

7. Pull the pinned image:

   ```bash
   docker compose pull
   ```

8. Deploy Tracktor:

   ```bash
   docker compose up -d
   ```

9. Verify container state and logs:

   ```bash
   docker compose ps
   docker compose logs --tail=100
   ```

10. Open the web interface and confirm that the existing vehicle data is present.

## Update Procedure

Tracktor is pinned to a specific image version, so upgrades are deliberate.

Before updating:

1. Review the upstream release notes for breaking changes or migration requirements.
2. Back up `/srv/docker/tracktor`.
3. Change the image tag in `/opt/stacks/tracktor/compose.yaml`.

Then:

```bash
cd /opt/stacks/tracktor
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Confirm that Tracktor loads normally and the existing data is intact.

## Security Considerations

Authentication is enabled:

```text
TRACKTOR_DISABLE_AUTH=false
```

The current deployment uses plain HTTP:

```text
HTTP_MODE=http
```

Tracktor therefore does not provide TLS encryption itself in the current configuration.

The container does not require privileged mode or Docker socket access.

If Tracktor is exposed through HTTPS later, `HTTP_MODE` should be reviewed because Tracktor uses it to determine whether authentication cookies receive the `secure` flag.

## Troubleshooting

### Web interface does not load

Check the container and logs:

```bash
cd /opt/stacks/tracktor
docker compose ps
docker compose logs --tail=100
```

Confirm that port `3333` is listening:

```bash
sudo ss -ltnp | grep ':3333 '
```

### Persistent data appears to be missing

Confirm the bind mount still exists in `compose.yaml`:

```text
/srv/docker/tracktor:/data
```

Inspect the host directory:

```bash
ls -lah /srv/docker/tracktor
```

Do not initialize a replacement database until the original data location has been identified.

## References

- https://github.com/javedh-dev/tracktor
- https://github.com/javedh-dev/tracktor/blob/dev/docs/installation.md
- https://github.com/javedh-dev/tracktor/blob/dev/docs/environment.md
