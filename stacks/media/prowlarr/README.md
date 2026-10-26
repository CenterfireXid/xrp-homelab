# Prowlarr

Prowlarr manages indexers for the XRP Homelab media stack and synchronizes them with Sonarr and Radarr.

> [!WARNING]
> **Legal notice:** This service is documented for educational and personal homelab use only. Use it only to manage, request, index, or download media that you legally own or are otherwise authorized to access. The operator is responsible for complying with applicable copyright laws, licenses, and service terms.

## Purpose

Prowlarr provides a single place to manage indexer configuration and synchronize supported indexers with Sonarr and Radarr.

The current deployment also uses a tag-scoped NordVPN SOCKS5 proxy for selected external indexer traffic.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Image | `lscr.io/linuxserver/prowlarr:2.6.5` |
| Installed version | `2.6.5.5623` |
| Container | `prowlarr` |
| Port | `9696` |
| Compose path | `/opt/stacks/prowlarr/compose.yaml` |
| Persistent data | `/srv/docker/prowlarr` |

## Architecture

```text
Prowlarr
├── Sonarr
├── Radarr
├── External indexers
│   └── NordVPN SOCKS5 when tagged "nord"
└── FlareSolverr
    └── Used when an indexer also requires the "flaresolverr" tag
```

Prowlarr, Sonarr, Radarr, and FlareSolverr are separate Compose projects. Local integrations use the Raspberry Pi host address and published service ports.

## Docker Compose

```yaml
services:
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:2.6.5
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Denver
    ports:
      - "9696:9696"
    volumes:
      - /srv/docker/prowlarr:/config
```

## Configuration

### Sonarr and Radarr

Both applications are configured in:

```text
Settings → Apps
```

Full Sync is used.

This is also where API keys are configured.

### NordVPN SOCKS5

The final deployment uses a **tag-scoped SOCKS5 Indexer Proxy**, not Prowlarr's global proxy.

Example configuration:

```text
Type: SOCKS5
Host: dallas.us.socks.nordhold.net
Port: 1080
Username: <NORD_SERVICE_USERNAME>
Password: <NORD_SERVICE_PASSWORD>
Tag: nord
```

Live NordVPN service credentials are stored only in the running application configuration.

All indexers will use NordVPN and will receive the `nord` tag.

### FlareSolverr

A separate FlareSolverr Indexer Proxy is configured using:

```text
http://192.168.30.115:8191
```

with the tag:

```text
flaresolverr
```

An indexer may have both:

```text
nord
flaresolverr
```

The tags do not serve the same purpose:

- `nord` selects the SOCKS5 egress proxy.
- `flaresolverr` enables FlareSolverr when the indexer requires challenge handling.

### Allowed Hosts

`Allowed Hosts` is hostname filtering, not a network allowlist.

The working local value for my deployment is:

```text
#PI IP
192.168.30.115
```

Do not enter a subnet such as:

```text
192.168.10.0
```

as though the field accepted CIDR/network ranges.

## Initial Deployment

Create the directories:

```bash
sudo mkdir -p /opt/stacks/prowlarr
sudo mkdir -p /srv/docker/prowlarr
sudo chown -R 1000:1000 /srv/docker/prowlarr
```

Place `compose.yaml` at:

```text
/opt/stacks/prowlarr/compose.yaml
```

Validate:

```bash
cd /opt/stacks/prowlarr
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

Open the UI, configure authentication, add Sonarr and Radarr, then configure the required indexers and indexer proxies.

## Access

```text
http://192.168.30.115:9696
```

Prowlarr is intended for private network access and is not exposed directly to the public Internet.

## Portainer

Portainer automatically detects the Compose-created `prowlarr` container.

Docker Compose remains the deployment source of truth. Portainer is used for status, logs, resource usage, restarts, and troubleshooting.

## Backup and Restore

Back up:

```text
/srv/docker/prowlarr
```

This directory contains the Prowlarr database, configuration, API keys, proxy credentials, and other application state.

Treat backups as sensitive.

For recovery:

1. Restore `/srv/docker/prowlarr`.
2. Restore `/opt/stacks/prowlarr/compose.yaml`.
3. Confirm ownership.
4. Run `docker compose config`.
5. Pull and deploy.
6. Test Sonarr, Radarr, indexers, and proxy routing.

## Update Procedure

Check Prowlarr and LinuxServer release notes before changing the pinned image tag.

```bash
cd /opt/stacks/prowlarr

docker compose config
docker compose pull
docker compose up -d

docker compose ps
docker compose logs --tail=100
```

Confirm application integrations and proxy tests after the update.

## Troubleshooting

### HTTP 400 / Invalid Hostname

Prowlarr returned HTTP 400 when `Allowed Hosts` was incorrectly configured as though it were a trusted network range.

Fix:

```text
Allowed Hosts: 192.168.30.115
```

Use `Trusted Networks` for forwarded client-network behavior when applicable. Do not use `Allowed Hosts` as a VLAN allowlist.

### Sonarr or Radarr fails with proxy tunnel errors

The original SOCKS5 configuration was applied as Prowlarr's global proxy.

This caused local requests to Sonarr and Radarr to enter the external SOCKS proxy path and generated errors similar to:

```text
An error occurred while establishing a connection to the proxy tunnel.
```

The final configuration uses a **tag-scoped Indexer Proxy** instead.

Local Sonarr and Radarr traffic therefore stays on the LAN.

### FlareSolverr receives Nord credentials from Prowlarr

With the global SOCKS5 proxy enabled, Prowlarr passed its Nord proxy configuration into FlareSolverr requests.

Chromium could not reliably use the authenticated Nord SOCKS5 connection directly and returned:

```text
net::ERR_SOCKS_CONNECTION_FAILED
```

The global proxy was removed.

The final design keeps Nord proxying tag-scoped inside Prowlarr and handles FlareSolverr's Nord egress separately through its local SOCKS relay.

### Intermittent indexer connectivity

An individual indexer may occasionally fail even when:

- DNS resolution succeeds.
- TLS connectivity from the container succeeds.
- The same indexer later passes its test.

Do not change global DNS, IPv6, TLS, or proxy settings based on a single intermittent failure.

Check the Prowlarr logs and reproduce the problem before changing the deployment.

## Security Considerations

Do not commit:

- NordVPN service credentials
- API keys
- application database files
- cookies or session data
- resolved configuration containing secrets

`/srv/docker/prowlarr` contains sensitive application state and should remain outside the repository.

## Roadmap

### Replace SOCKS5 with VPN-based egress

The current NordVPN SOCKS5 configuration works, but it is application-level proxying rather than network-level isolation.

A future revision should evaluate moving selected media-stack traffic behind a dedicated VPN container or network namespace.

Goals:

- network-level encrypted egress
- fail-closed behavior if the VPN disconnects
- fewer application-specific proxy settings
- simpler credential management
- consistent routing between Prowlarr, FlareSolverr, and other selected services

The current SOCKS5 configuration should remain in place until the VPN design is tested and verified without disrupting local service communication.

## References

- Prowlarr: https://prowlarr.com/
- Prowlarr GitHub: https://github.com/Prowlarr/Prowlarr
- LinuxServer Prowlarr: https://docs.linuxserver.io/images/docker-prowlarr/
- NordVPN SOCKS5 Proxy List: https://support.nordvpn.com/hc/en-us/sections/24555410533905-SOCKS5-Proxy
