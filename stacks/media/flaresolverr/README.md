# FlareSolverr

FlareSolverr provides challenge handling for selected Prowlarr indexers.

The XRP Homelab deployment routes FlareSolverr browser traffic through a local GOST SOCKS5 relay, which authenticates upstream to NordVPN.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| FlareSolverr image | `ghcr.io/flaresolverr/flaresolverr:v3.5.2` |
| Relay image | `ginuerzh/gost:2.12.0` |
| Port | `8191` |
| Compose path | `/opt/stacks/flaresolverr/compose.yaml` |
| Persistent data | `/srv/docker/flaresolverr` |

## Architecture

```text
Prowlarr
  ↓
FlareSolverr
  ↓
GOST SOCKS5 relay
  ↓
NordVPN SOCKS5
```

The GOST relay is not published to the host and is only reachable on the Compose network.

## Configuration

Create `/opt/stacks/flaresolverr/.env` from `.env.example`:

```dotenv
NORD_SOCKS_HOST=dallas.us.socks.nordhold.net
NORD_SOCKS_PORT=1080
NORD_SOCKS_USERNAME=<NORD_SERVICE_USERNAME>
NORD_SOCKS_PASSWORD=<NORD_SERVICE_PASSWORD>
```

Protect it by making it only readable/writeable by owner:

```bash
chmod 600 /opt/stacks/flaresolverr/.env
```

Prowlarr uses:

```text
http://192.168.30.115:8191
```

with the tag:

```text
flaresolverr
```

## Initial Deployment

```bash
sudo mkdir -p /opt/stacks/flaresolverr /srv/docker/flaresolverr
sudo chown -R 1000:1000 /srv/docker/flaresolverr

cd /opt/stacks/flaresolverr
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
curl http://127.0.0.1:8191/health
```

Expected health response:

```json
{"status":"ok"}
```
> [!WARNING]
>`docker compose config` resolves values from `.env` and can reveal secrets.

### Prowlarr Integration

FlareSolverr is configured in Prowlarr as an Indexer Proxy.

In Prowlarr, open:

```text
Settings → Indexer Proxies → + → FlareSolverr
```

Configure:

```text
Name: FlareSolverr
Host: http://192.168.30.115:8191
Tags: flaresolverr
```

Save and test the proxy.

Assign the matching tag to any indexer that requires FlareSolverr:

```text
flaresolverr
```

If the same indexer should also use the NordVPN SOCKS5 Indexer Proxy configured in Prowlarr, assign both tags:

```text
nord
flaresolverr
```

The tags serve different purposes:

- `nord` selects Prowlarr's tag-scoped NordVPN SOCKS5 Indexer Proxy.
- `flaresolverr` enables the FlareSolverr Indexer Proxy for challenge handling.

Do not configure NordVPN SOCKS5 as Prowlarr's global proxy. During testing, the global proxy caused local requests to Sonarr, Radarr, and FlareSolverr to be routed through the external SOCKS5 proxy.

FlareSolverr handles its own NordVPN egress independently:

```text
Prowlarr
  ↓
http://192.168.30.115:8191
  ↓
FlareSolverr
  ↓
socks5://nord-socks-relay:1080
  ↓
GOST
  ↓
Authenticated NordVPN SOCKS5
```

This keeps the Prowlarr-to-FlareSolverr API request on the local network while FlareSolverr's browser traffic exits through NordVPN.

#### Verify

After saving the FlareSolverr Indexer Proxy, use Prowlarr's built-in **Test** button.

Then test an indexer carrying the `flaresolverr` tag.

On the Docker host, review both services:

```bash
cd /opt/stacks/flaresolverr

docker compose logs --since=5m flaresolverr
docker compose logs --since=5m nord-socks-relay
```

A successful request should show Prowlarr reaching FlareSolverr locally and FlareSolverr using the internal GOST relay for external browser traffic.

## Troubleshooting

### `ERR_SOCKS_CONNECTION_FAILED`

Direct authenticated NordVPN SOCKS5 access from Chromium failed with:

```text
net::ERR_SOCKS_CONNECTION_FAILED
```

The working design uses an unauthenticated local SOCKS5 listener in GOST. GOST then authenticates upstream to NordVPN.

### `socks5h://` appears successful but bypasses NordVPN

A `socks5h://` test returned the normal residential public address instead of the NordVPN exit.

A successful HTTP response is not enough to prove proxying. Verify the egress address independently.

### Prowlarr passes Nord credentials into FlareSolverr

This occurred when NordVPN SOCKS5 was configured as Prowlarr's global proxy.

The final Prowlarr configuration uses a tag-scoped SOCKS5 Indexer Proxy. FlareSolverr handles its own NordVPN egress through GOST.

### Intermittent challenge failures

Inspect both containers:

```bash
docker compose logs --since=5m flaresolverr
docker compose logs --since=5m nord-socks-relay
```

Look for Chromium proxy errors, challenge failures, or upstream relay connection errors before changing DNS or network settings.

## Backup and Restore

Back up:

```text
/srv/docker/flaresolverr
/opt/stacks/flaresolverr/.env
```

The `.env` file contains credentials and must be stored securely.

For restore, restore the data directory and `.env`, restore `compose.yaml`, validate with `docker compose config`, then pull and deploy.

## Roadmap

Replace application-level SOCKS5 routing with dedicated VPN-based egress.

Goals:

- network-level encrypted routing
- fail-closed behavior
- centralized credential management
- consistent egress for selected services
- continued direct access to local services

Keep the current SOCKS5 relay until the VPN replacement is tested and verified.

## References

- FlareSolverr: https://github.com/FlareSolverr/FlareSolverr
- GOST: https://github.com/ginuerzh/gost
- NordVPN: https://nordvpn.com/
