# Mealie Migration Notes

This document records the migration of Mealie from TrueNAS to `xrp-homelab`.

It is intended as a historical record and recovery reference. The main deployment instructions are in [`README.md`](./README.md).

## Source Environment

The original Mealie deployment ran on TrueNAS with:

| Setting | Value |
|---|---|
| Mealie version | `v3.8.0` |
| Database | PostgreSQL |
| Application mode | Production |
| Default group | `Home` |
| Default household | `Family` |

The source contained:

- Existing users
- Recipes
- Recipe images
- Historical meal / recipe data
- Previous "made" dates and other historical data

## Migration Goal

Move Mealie to a Raspberry Pi 5 while preserving all application data and simplifying the database layer.

Target:

| Setting | Value |
|---|---|
| Host | `xrp-homelab` |
| Platform | Raspberry Pi 5 / ARM64 |
| Initial Mealie version | `v3.8.0` |
| Final Mealie version | `v3.28.0` |
| Database | SQLite |
| Persistent data | `/srv/docker/mealie/data` |
| Live Compose | `/opt/stacks/mealie/compose.yaml` |
| Host port | `9925` |

## Migration Strategy

The migration intentionally avoided manually copying PostgreSQL files or selectively copying Mealie directories.

Instead:

```text
TrueNAS
Mealie v3.8.0
PostgreSQL
      |
      | Mealie native backup
      v
Raspberry Pi
Mealie v3.8.0
SQLite
      |
      | verification
      v
Raspberry Pi
Mealie v3.28.0
SQLite
```

Using the same Mealie application version for the initial restore separated the migration from the later application upgrade.

## Procedure Used

### 1. Create Native Backup on TrueNAS

A Mealie backup ZIP was created from the Mealie web UI running on the TrueNAS.

The backup included the database and data directory needed for application-level restore.

### 2. Deploy Mealie v3.8.0 on Raspberry Pi

The initial Pi deployment used:

```yaml
image: ghcr.io/mealie-recipes/mealie:v3.8.0
```

with:

```yaml
DB_ENGINE: sqlite
```

Persistent storage:

```text
/srv/docker/mealie/data:/app/data
```

### 3. Transfer Backup to Pi

The browser upload did not successfully initiate the backup upload due to me completing this remotely using my hotspot.

The backup ZIP was therefore transferred directly to:

```text
/srv/docker/mealie/data/backups/
```

using `scp`.

Example:

```bash
scp "/path/to/mealie_backup.zip" \
  xidney@<xrp-homelab-ip>:/srv/docker/mealie/data/backups/
```

The backup then appeared in the Mealie backup UI, however I took a break as I was frustrated with the slow connection.

Once reliable internet connectivity was restored, the native backup upload function on the web UI was used `Settings > Admin Settings > Backups` 

### 4. Restore Backup

The backup was restored from Mealie's backup page.

The restore replaced the fresh SQLite application data with the source Mealie data.

After restore:

- Existing credentials worked
- Recipes were present
- Historical data was present
- Images were present
- Users were restored

### 5. Verify SQLite

Environment:

```bash
docker inspect mealie \
  --format '{{range .Config.Env}}{{println .}}{{end}}' \
  | grep '^DB_ENGINE='
```

Expected:

```text
DB_ENGINE=sqlite
```

Database file:

```bash
file /srv/docker/mealie/data/mealie.db
```

The file was identified as an SQLite 3.x database.

Logs also confirmed:

```text
Database connection established.
Context impl SQLiteImpl.
```

### 6. Create Post-Migration Backup

A new native Mealie backup was created from the Pi after the restore was verified.

### 7. Upgrade to v3.28.0

Before upgrading, a cold archive was created.

The image tag was changed from:

```text
v3.8.0
```

to:

```text
v3.28.0
```

The new image was pulled and started.

Startup logs showed database migrations completing successfully, followed by:

```text
Application startup complete.
```

The container returned to:

```text
healthy
```

after both initial startup and a manual restart.

## Current State

Current deployment:

```text
Host:       xrp-homelab
Mealie:     v3.28.0
Database:   SQLite
Host port:  9925
Data:       /srv/docker/mealie/data
Compose:    /opt/stacks/mealie/compose.yaml
```

Current persistent mount:

```text
/srv/docker/mealie/data -> /app/data
```

## Lessons Learned

### Keep Migration and Upgrade Separate

Restore onto the same application version first, verify the data, and only then upgrade.

This makes troubleshooting much easier because a failed restore can be separated from a failed schema/application upgrade.

### Do Not Assume Browser Upload Is Required

A native backup ZIP can be placed directly into:

```text
/srv/docker/mealie/data/backups/
```

when the browser upload path is unreliable due to internet speed or otherwise.

### Preserve a Cold Backup Before Major Upgrades

Changing the Docker image tag back may not be enough after database schema migrations.

A cold backup should preserve both:

```text
/opt/stacks/mealie/compose.yaml
/srv/docker/mealie/data
```

### Pin Image Versions

Use explicit versions such as:

```yaml
image: ghcr.io/mealie-recipes/mealie:v3.28.0
```

instead of:

```yaml
image: ghcr.io/mealie-recipes/mealie:latest
```

This keeps upgrades intentional and reproducible.
