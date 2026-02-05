# Speedtest Tracker

## Use Case
I added Speedtest Tracker to continuously verify that my ISP delivers the speeds I’m paying for, and to build a historical record of performance over time. This helps me:
- Identify slowdowns and patterns (time-of-day congestion, intermittent drops)
- Validate improvements after network changes (router/AP placement, DNS changes, etc.)
- Gather evidence if I need to dispute service quality with my ISP

## What it does
- Runs automated internet speed tests on a schedule
- Stores results over time (download, upload, ping, jitter depending on config)
- Provides a dashboard to visualize trends and history

## Access
- **LAN:** `http://<PI_LAN_IP>:8080`
- **Tailscale:** `http://<PI_TAILSCALE_IP>:8080`

## Ports
- Container: `80`
- Host: `8080`

## Storage
- Persistent data: `/srv/docker/speedtest-tracker`
- Database: **SQLite** (stored in the mounted config directory)

## Deployment Method
- Deployed as a **Portainer Stack** using Docker Compose.
- Data is stored outside the container to survive updates and reboots.

## Lessons Learned

- **PUID/PGID mapping (permissions):** Confirmed the correct `PUID` and `PGID` by running:
  ```bash
  id $USER
  ```
  On this system, both values are `1000`. These values make the container write files as the same user/group as the host, preventing permission issues on bind mounts.

- **Pre-create storage directory (avoid root-owned folders):** Created the persistent directory before deploying:
  ```bash
  sudo mkdir -p /srv/docker/speedtest-tracker
  sudo chown -R $USER:$USER /srv/docker/speedtest-tracker
  ```
  Doing this prevents Docker from auto-creating the folder as `root`, which can cause read/write errors and makes maintenance/backups harder.

- **APP_KEY generation (encryption purpose):** Generated the application encryption key using:
  ```bash
  echo -n 'base64:'; openssl rand -base64 32
  ```
  `APP_KEY` is required to encrypt sensitive app data (sessions/cookies and other stored values). Each deployment should have a unique key.

- **Persistence check:** Verified SQLite + `/config` bind mount persists data across container restarts and host reboots.

- **Scheduling speed tests via cron expression:** Speedtest Tracker schedules automated tests using the `SPEEDTEST_SCHEDULE` environment variable in cron format. Times are based on the set container timezone. Changes take effect after updating/redeploying the Portainer stack. Examples:
  ```text
  # Once per day (midnight)
  SPEEDTEST_SCHEDULE=0 0 * * *

  # Once per day at 3:00 AM
  SPEEDTEST_SCHEDULE=0 3 * * *

  # Every 2 hours (runs at 00:00, 02:00, 04:00, ...)
  SPEEDTEST_SCHEDULE=0 */2 * * *

  # Every 2 hours starting at 1:00 AM (runs at 01:00, 03:00, 05:00, ...)
  SPEEDTEST_SCHEDULE=0 1-23/2 * * *

- Breakdown (5-field cron: minute hour day-of-month month day-of-week):
  * **Minute:** `0` &rarr; Specifies the job runs at the beginning of the hour (minute 0).
  * **Hour:** `1-23/2` &rarr; The range `1-23` means every hour from 1 AM to 11 PM. The step value `/2` means it runs every second hour within that range. This results in runs at 1 AM, 3 AM, 5 AM, and so on, every odd hour.
  * **Day of the Month:** `*` &rarr; The job runs every day of the month.
  * **Month:** `*` &rarr; This job runs every month.
  * **Day of the Week:** `*` &rarr; The job runs every day of the week.


