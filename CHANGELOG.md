# Changelog

Notable deployment, configuration, and documentation changes to XRP Homelab are recorded here. Minor editorial fixes do not require a changelog entry.

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
