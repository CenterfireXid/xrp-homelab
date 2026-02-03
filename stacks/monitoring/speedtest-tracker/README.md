# Speedtest Tracker

## Use Case
I added Speedtest Tracker to continuously verify that my ISP delivers the speeds I’m paying for, and to build a historical record of performance over time. This helps me:
- identify slowdowns and patterns (time-of-day congestion, intermittent drops)
- validate improvements after network changes (router/AP placement, DNS changes, etc.)
- gather evidence if I need to dispute service quality with my ISP

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
- Scheduling speed tests is done via an environment variable, not the UI.
- Added `SPEEDTEST_SCHEDULE=0 3 * * *` to the Docker environment variables to run tests daily at 3:00 AM.
- Changes require updating/redeploying the Portainer stack to take effect.
