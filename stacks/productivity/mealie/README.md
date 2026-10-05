# Mealie

Self-hosted recipe manager deployed on `xrp-homelab` using Docker Compose.

This deployment runs Mealie on a Raspberry Pi 5 with SQLite-backed persistent storage.

## Purpose

Mealie provides a centralized household recipe manager with support for:

- Recipes and recipe images
- Users and households
- Meal planning
- Historical recipe data
- Backup and restore
- Recipe scraping/import
- Tags, categories, favorites, and other recipe metadata

### Current Deployment

<img width="1416" height="760" alt="My Mealie recipe dashboard showing several saved recipes" src="https://github.com/user-attachments/assets/ada27167-2322-4d59-80c8-cb0cff93049e" />

<p align="center">
  <em>My recipe dashboard from the current XRP-HOMELAB Mealie deployment.</em>
</p>

<p align="center">
<img width="246" height="370" alt="Mealie navigation showing recipes, meal planner, shopping lists, timeline, and other features" src="https://github.com/user-attachments/assets/714e4e0e-5a12-44cf-a170-93a4b168b446" />
</p>

<p align="center">
  <em>Mealie feature navigation, including Recipes, Meal Planner, Shopping Lists, and Timeline.</em>
</p>

## Architecture

| Component | Value |
|---|---|
| Host | Raspberry Pi 5 |
| Hostname | `xrp-homelab` |
| Architecture | ARM64 |
| Container | `mealie` |
| Image | `ghcr.io/mealie-recipes/mealie:v3.28.0` |
| Database | SQLite |
| Host port | `9925` |
| Container port | `9000` |
| Restart policy | `unless-stopped` |
| PUID / PGID | `1000 / 1000` |
| Time zone | `America/Denver` |

The live Docker Compose definition is stored separately from persistent application data:

```text
/opt/stacks/mealie/
└── compose.yaml

/srv/docker/mealie/
└── data/
    ├── backups/
    ├── groups/
    ├── mealie.db
    ├── recipes/
    ├── templates/
    ├── users/
    └── ...
```

## Repository Structure

This service is documented in the homelab repository under:

```text
stacks/
└── productivity/
    └── mealie/
        ├── README.md
        ├── MIGRATION.md
        └── compose.yaml
```

The repository copy is for documentation and reproducibility. Docker runs the live stack from `/opt/stacks/mealie/compose.yaml`.

## Configuration

Create the live stack directory:

```bash
sudo mkdir -p /opt/stacks/mealie
sudo chown -R "$USER":"$USER" /opt/stacks/mealie
```

Create the persistent data directory:

```bash
sudo mkdir -p /srv/docker/mealie/data
sudo chown -R 1000:1000 /srv/docker/mealie
```

Create `/opt/stacks/mealie/compose.yaml`:

```yaml
services:
  mealie:
    image: ghcr.io/mealie-recipes/mealie:v3.28.0
    container_name: mealie
    restart: unless-stopped

    ports:
      - "9925:9000"

    volumes:
      - /srv/docker/mealie/data:/app/data

    environment:
      ALLOW_SIGNUP: "false"
      PUID: 1000
      PGID: 1000
      TZ: America/Denver
      DB_ENGINE: sqlite
```

Validate the Compose configuration before deployment:

```bash
cd /opt/stacks/mealie
docker compose config
```

The expected Compose project/network name is:

```text
mealie_default
```

## Initial Deployment

Start the stack:

```bash
cd /opt/stacks/mealie
docker compose up -d
```

Check container state:

```bash
docker compose ps
```

Follow startup logs:

```bash
docker compose logs -f --tail=150 mealie
```

Press `Ctrl+C` to stop following logs without stopping the container.

Verify the persistent mount:

```bash
docker inspect mealie \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected:

```text
/srv/docker/mealie/data -> /app/data
```

Verify SQLite:

```bash
docker inspect mealie \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep '^DB_ENGINE='
```

Expected:

```text
DB_ENGINE=sqlite
```

Verify the database file:

```bash
file /srv/docker/mealie/data/mealie.db
```

Expected output should identify it as an SQLite 3 database.

## Access

Mealie is exposed on host port `9925`:

```text
http://<xrp-homelab-ip>:9925
```

The container listens internally on port `9000`.

## Portainer

This stack may be viewed and monitored from Portainer, but the Compose file under `/opt/stacks/mealie/compose.yaml` is the source of truth.

Avoid making configuration-only changes directly in Portainer unless the same change is also reflected in the Compose file.

## Persistent Storage

All persistent Mealie application data is stored under:

```text
/srv/docker/mealie/data
```

Important paths include:

```text
/srv/docker/mealie/data/mealie.db
/srv/docker/mealie/data/backups/
/srv/docker/mealie/data/recipes/
/srv/docker/mealie/data/users/
```

Because SQLite is used, `mealie.db` contains the application database.

Do not delete `/srv/docker/mealie/data` when recreating or upgrading the container.

## Backup

### Native Mealie Backup

Use Mealie's built-in backup feature from the web UI.

Native backup files are stored under:

```text
/srv/docker/mealie/data/backups/
```

These backups are preferred for normal Mealie restore/migration workflows.

Keep at least one copy outside the Pi.

### Cold Filesystem Backup

Before major upgrades, create a stopped/cold backup.

Stop Mealie:

```bash
cd /opt/stacks/mealie
docker compose stop
```

Create the backup directory:

```bash
sudo mkdir -p /srv/backups/mealie
```

Back up both the Compose definition and application data:

```bash
sudo tar -czf \
  /srv/backups/mealie/mealie-pre-upgrade-$(date +%Y-%m-%d-%H%M%S).tar.gz \
  /opt/stacks/mealie/compose.yaml \
  /srv/docker/mealie/data
```

Restart Mealie if an upgrade is not being performed immediately:

```bash
docker compose up -d
```

List backups:

```bash
ls -lh /srv/backups/mealie/
```

## Restore

### Native Mealie Restore

For a normal application-level restore:

1. Deploy Mealie.
2. Open the Mealie backup page.
3. Upload or place the backup ZIP into:
   ```text
   /srv/docker/mealie/data/backups/
   ```
4. Refresh the backup page.
5. Select **Backup Restore**.
6. Confirm the destructive restore warning.
7. Log back in with the restored user account.
8. Verify recipes, images, users, meal history, and other expected data.

If browser upload fails, copy the ZIP directly to the backup directory with `scp`:

```bash
scp /path/to/mealie-backup.zip \
  username@<xrp-homelab-ip>:/srv/docker/mealie/data/backups/
```

### Cold Filesystem Restore

Use this only when rolling back the complete application state.

Stop the stack:

```bash
cd /opt/stacks/mealie
docker compose down
```

Preserve the current data before restoring:

```bash
sudo mv /srv/docker/mealie/data \
  /srv/docker/mealie/data.failed-$(date +%Y-%m-%d-%H%M%S)
```

Extract the selected cold backup from `/`:

```bash
cd /
sudo tar -xzf /srv/backups/mealie/<backup-file>.tar.gz
```

Validate the restored Compose file:

```bash
cd /opt/stacks/mealie
docker compose config
```

Start Mealie:

```bash
docker compose up -d
docker compose ps
```

Do not delete the preserved failed-state directory until the rollback has been verified.

## Updating Mealie

Mealie is intentionally pinned to a specific image version instead of `latest`.

Current pinned image:

```text
ghcr.io/mealie-recipes/mealie:v3.28.0
```

### Safe Upgrade Procedure

1. Create a fresh native Mealie backup.
2. Create a cold filesystem backup.
3. Stop Mealie if not already stopped:
   ```bash
   cd /opt/stacks/mealie
   docker compose stop
   ```
4. Edit the image tag in:
   ```text
   /opt/stacks/mealie/compose.yaml
   ```
5. Validate:
   ```bash
   docker compose config
   docker compose config | grep image:
   ```
6. Pull the new image:
   ```bash
   docker compose pull
   ```
7. Recreate/start the container:
   ```bash
   docker compose up -d
   ```
8. Watch migration/startup logs:
   ```bash
   docker compose logs -f --tail=150 mealie
   ```
9. Verify:
   ```bash
   docker compose ps
   ```
10. Confirm the web UI, user login, recipes, images, history, and backups.
11. Restart once:
   ```bash
   docker compose restart
   ```
12. Confirm the container returns to `healthy`.
13. Create a new native backup after the upgrade succeeds.
14. Update the GitHub copy of `compose.yaml` and documentation.

### Version Pinning

Use:

```yaml
image: ghcr.io/mealie-recipes/mealie:vX.Y.Z
```

Avoid:

```yaml
image: ghcr.io/mealie-recipes/mealie:latest
```

Pinning prevents an unexpected application upgrade during an unrelated pull or redeployment.

## Rollback

If an upgrade fails after database migrations have run, do not assume that changing the image tag back is sufficient.

Use the pre-upgrade cold backup to restore both:

```text
/opt/stacks/mealie/compose.yaml
/srv/docker/mealie/data
```

This restores the application definition and database/data directory to the same pre-upgrade point.

## Migration Notes

This deployment was migrated from a TrueNAS-hosted Mealie instance running:

```text
Mealie v3.8.0
Database: PostgreSQL
```

The migration used Mealie's native backup/restore workflow and restored into:

```text
Mealie v3.8.0
Database: SQLite
Host: Raspberry Pi 5
```

The restored installation was verified before being upgraded to `v3.28.0`.

Detailed migration notes are available in [`MIGRATION.md`](./MIGRATION.md).

## Security Considerations

- Public user signup is disabled with `ALLOW_SIGNUP=false`.
- Keep Mealie and its dependencies updated.
- Do not expose port `9925` directly to the public Internet without an appropriate reverse proxy/authentication design.
- Keep backup copies outside the Pi.
- Treat Mealie backup ZIPs and `mealie.db` as sensitive because they contain application/user data.
- Do not commit persistent data, backups, secrets, or SQLite database files to GitHub.

## Troubleshooting

### Container Health

```bash
cd /opt/stacks/mealie
docker compose ps
```

### Logs

```bash
docker compose logs --tail=200 mealie
```

### Follow Logs

```bash
docker compose logs -f --tail=150 mealie
```

### Check SQLite Configuration

```bash
docker inspect mealie \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep '^DB_ENGINE='
```

### Check Database Type

```bash
file /srv/docker/mealie/data/mealie.db
```

### Check Persistent Mount

```bash
docker inspect mealie \
  --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

### Check Port Binding

```bash
sudo ss -ltnp | grep ':9925 '
```

### Check Backup Directory

```bash
ls -lah /srv/docker/mealie/data/backups/
```

### Validate Compose

```bash
cd /opt/stacks/mealie
docker compose config
```

## References

- Mealie project: https://github.com/mealie-recipes/mealie
- Mealie documentation: https://docs.mealie.io/
- Mealie container registry: `ghcr.io/mealie-recipes/mealie`

