# Changelog

Notable deployment, configuration, and documentation changes to XRP Homelab are recorded here. Minor editorial fixes do not require a changelog entry.

## 2026-10-26

### Added

- Added the Prowlarr Docker Compose stack under `stacks/media/prowlarr`.
- Documented the running Prowlarr `v2.6.5.5623` deployment using the pinned LinuxServer image `lscr.io/linuxserver/prowlarr:2.6.5`.
- Added persistent Prowlarr application data under `/srv/docker/prowlarr`.
- Added Sonarr and Radarr application integrations using Full Sync.
- Added tag-scoped NordVPN SOCKS5 proxying for selected indexer traffic.
- Added FlareSolverr integration using a dedicated `flaresolverr` tag.

### Documentation

- Added the reproducible Prowlarr initial deployment, verification, backup, restore, update, security, and troubleshooting procedures.
- Documented the corrected `Allowed Hosts` configuration after resolving the HTTP `400 Invalid Hostname` issue.
- Documented the move away from Prowlarr's global proxy after it interfered with local Sonarr, Radarr, and FlareSolverr communication.
- Added `NORDVPN-SOCKS5.md` with sanitized NordVPN SOCKS5 configuration, verification, credential-handling, and troubleshooting procedures.
- Documented the use of separate `nord` and `flaresolverr` tags for proxy egress and challenge handling.
- Added a roadmap to replace application-level SOCKS5 proxying with dedicated VPN-based egress and fail-closed routing.

## 2026-10-15

### Added

- Added the qBittorrent Docker Compose stack under `stacks/media/qbittorrent`.
- Documented the running qBittorrent `v5.2.3` deployment using the pinned official image `qbittorrentofficial/qbittorrent-nox:5.2.3-1`.
- Added persistent qBittorrent configuration under `/srv/docker/qbittorrent/config`.
- Added NFS-backed download storage from `/mnt/unas-media/downloads`.
- Configured separate incomplete and complete download paths.
- Configured authenticated NordVPN SOCKS5 proxying for BitTorrent traffic.

### Documentation

- Added the reproducible qBittorrent initial deployment, verification, backup, restore, update, security, and troubleshooting procedures.
- Documented torrent-address and proxy-failure verification of the SOCKS5 configuration.
- Documented the observed UDP tracker limitation through the current SOCKS5 configuration.
- Documented defensive file-type exclusions for executable and script content.
- Documented the expected NFS ownership warning and verified download-directory write behavior.

## 2026-10-08

### Added

- Added the Homarr Docker Compose stack under `stacks/management/homarr`.
- Documented the running Homarr `v1.77.2` deployment on host port `7575`.
- Added persistent Homarr application data under `/srv/docker/homarr`.
- Added a restricted LinuxServer Docker socket proxy for container discovery, logs, and start/stop/restart controls.
- Added `.env.example` for the required Homarr encryption key.

### Documentation

- Added the reproducible Homarr initial deployment and verification procedure.
- Documented the private `docker-api` network and restricted socket-proxy permissions.
- Documented the observed Redis `vm.overcommit_memory` startup warning without changing the host sysctl setting.

## 2026-10-05

### Documentation

- Added the Mealie migration procedure.
- Documented the TrueNAS PostgreSQL to Raspberry Pi SQLite migration workflow.

## 2026-10-04

### Added

- Added the Mealie Docker Compose stack under `stacks/productivity/mealie`.
- Deployed Mealie `v3.28.0` using the pinned official image `ghcr.io/mealie-recipes/mealie:v3.28.0`.
- Added persistent Mealie application data under `/srv/docker/mealie/data`.
- Configured Mealie to use SQLite with host port `9925`.

### Documentation

- Added the Mealie deployment and operations guide.
- Documented initial deployment, verification, backup, restore, update, rollback, security, and troubleshooting procedures.

## 2026-09-21

### Documentation

- Added the Jellyfin deployment and operations guide.
- Documented initial deployment, verification, backup, restore, update, security, and troubleshooting procedures.
- Added the Jellyfin architecture diagram and storage layout.

## 2026-09-20

### Added

- Deployed Jellyfin 12.0.0 using the pinned official image `jellyfin/jellyfin:12.0.20260908-012347`.
- Added the Jellyfin Docker Compose definition under `stacks/media/jellyfin`.
- Added persistent Jellyfin configuration under `/srv/docker/jellyfin/config` and cache storage under `/srv/docker/jellyfin/cache`.
- Added NFS-backed media mounts for movies, TV shows, kids shows, and locally stored YouTube media.
- Added Jellyfin to the repository's current infrastructure list.

## 2026-09-13

### Documentation

- Added the Tracktor deployment procedure.

## 2026-09-06

### Added

- Added the Tracktor Docker Compose stack under `stacks/productivity/tracktor`.

### Documentation

- Added Tracktor deployment documentation.
- Updated repository documentation.

## 2026-07-22

### Documentation

- Updated repository documentation.

## 2026-02-05

### Changed

- Updated the Speedtest Tracker Docker Compose configuration.

### Documentation

- Updated repository documentation.

## 2026-02-03

### Added

- Added the sanitized Speedtest Tracker Docker Compose configuration under `stacks/monitoring/speedtest-tracker`.
- Added README documentation for Speedtest Tracker and the repository.

### Documentation

- Expanded the Speedtest Tracker README with deployment notes and lessons learned.
