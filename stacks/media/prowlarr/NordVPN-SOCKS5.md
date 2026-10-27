# NordVPN SOCKS5 for Prowlarr

This document describes the current NordVPN SOCKS5 configuration used by Prowlarr on XRP Homelab.

The proxy is applied only to explicitly tagged indexers. It is not configured as Prowlarr's global proxy.

## Purpose

The SOCKS5 proxy provides application-level alternate egress for selected Prowlarr indexer traffic.

Current flow:

```text
Prowlarr
  │
  ├── Local Sonarr/Radarr communication
  │      └── Direct LAN
  │
  └── Indexer tagged "nord"
         └── NordVPN SOCKS5
```

This distinction is important because local application traffic should not be sent through the external SOCKS5 proxy.

## Credentials

Use [NordVPN service credentials](https://support.nordvpn.com/hc/en-us/articles/19685514639633-How-to-find-your-service-credentials-for-manual-NordVPN-setup) rather than your actual account login credentials.

Prowlarr stores the configured proxy credentials inside its persistent application state:

```text
/srv/docker/prowlarr
```

Treat backups of this directory as sensitive.

## Configure the SOCKS5 Indexer Proxy

In Prowlarr:

```text
Settings → Indexer Proxies → + → SOCKS5
```

Configure:

```text
Name: NordVPN SOCKS5
Host: dallas.us.socks.nordhold.net
Port: 1080
Username: <NORD_SERVICE_USERNAME>
Password: <NORD_SERVICE_PASSWORD>
Tags: nord
```

Save and test the proxy.

Prowlarr applies an Indexer Proxy only to indexers sharing the same tag.

Assign:

```text
nord
```

only to indexers that should use the NordVPN SOCKS5 connection, in my case, all indexers will utilize the proxy.

## Why the Global Proxy Is Not Used

The first configuration used Prowlarr's global proxy under the General settings.

That caused Prowlarr to attempt to route local application traffic through NordVPN, including requests to Sonarr, Radarr, and FlareSolverr.

The resulting logs included errors such as:

```text
An error occurred while establishing a connection to the proxy tunnel.
```

The corrected design removes the global proxy and uses a tag-scoped Indexer Proxy instead.

This keeps local application integrations direct while proxying only selected external requests.

## Verify Proxy Operation

Test the SOCKS5 proxy from Prowlarr using its built-in Test function.

Then test a tagged indexer.

Review recent Prowlarr logs:

```bash
cd /opt/stacks/prowlarr

docker compose logs --since=5m \
  | grep -Ei 'proxy|indexer|error|exception'
```

A successful indexer test confirms application-level connectivity through the configured proxy.

## Independent SOCKS5 Test

The NordVPN endpoint can also be tested independently from the host.

Load credentials from a local shell environment or other protected source rather than putting them directly into command history.

Example:

```bash
curl --socks5-hostname "<NORD_SOCKS_HOST>:1080" \
  --proxy-user "<NORD_SERVICE_USERNAME>:<NORD_SERVICE_PASSWORD>" \
  https://api.ipify.org
```

Compare the returned address with a direct request:

```bash
curl https://api.ipify.org
```

The addresses should differ when SOCKS5 proxying is functioning.

## FlareSolverr Interaction

An indexer may have both:

```text
nord
flaresolverr
```

These tags perform different functions.

`nord` selects the Prowlarr SOCKS5 Indexer Proxy.

`flaresolverr` selects the FlareSolverr Indexer Proxy when challenge handling is required.

The FlareSolverr container does not use Prowlarr's Nord credentials directly in the final design. Its NordVPN egress is handled through a separate local SOCKS5 relay documented with the FlareSolverr deployment.

## Troubleshooting

### Local Sonarr or Radarr integration fails

**Symptom**

Prowlarr logs:

```text
An error occurred while establishing a connection to the proxy tunnel.
```

while communicating with Sonarr or Radarr.

**Cause**

The Nord SOCKS5 connection was configured as Prowlarr's global proxy.

**Fix**

Remove the global proxy and use a tag-scoped SOCKS5 Indexer Proxy instead.

Verify that local Sonarr and Radarr application tests succeed afterward.

### FlareSolverr receives Nord username and password

**Symptom**

FlareSolverr request logs contain a proxy object pointing directly to the Nord SOCKS5 endpoint with username and password values.

**Cause**

Prowlarr's global proxy configuration was being passed into FlareSolverr requests.

**Fix**

Remove the Prowlarr global proxy.

Use:

```text
NordVPN SOCKS5 → tag: nord
FlareSolverr    → tag: flaresolverr
```

The FlareSolverr deployment handles its own Nord egress separately.

### SOCKS connection works intermittently

First confirm the problem is actually proxy-related.

Test:

1. The SOCKS5 Indexer Proxy in Prowlarr.
2. The affected indexer.
3. The Nord endpoint independently with `curl`.
4. Prowlarr logs for tunnel or timeout errors.

Do not change DNS, IPv6, or the entire Compose deployment based on a single intermittent indexer failure.

## Security Notes

SOCKS5 is being used here as application-level proxying.

It does not provide the same isolation as placing the container behind a VPN network namespace with a kill switch.

Important limitations:

- only applications explicitly configured to use the proxy are affected
- incorrectly tagged or unconfigured traffic may use the normal host route
- failure behavior depends on the application
- SOCKS5 itself is not a substitute for network-level routing policy

The current setup is therefore treated as an interim solution.

## Roadmap

### Replace SOCKS5 with VPN-Based Egress

Future work should evaluate moving selected media services behind a dedicated VPN container or VPN network namespace.

Preferred goals:

```text
Prowlarr
FlareSolverr
qBittorrent
        │
        ▼
Dedicated VPN egress
        │
        ├── encrypted tunnel
        └── fail-closed routing
```

The replacement should provide:

- network-level VPN routing
- fail-closed behavior if the tunnel drops
- consistent external egress
- fewer per-application proxy settings
- centralized VPN credential management
- continued direct LAN access to Sonarr, Radarr, and other internal services

A future implementation may evaluate a dedicated VPN container such as Gluetun, but no migration should occur until the routing model is tested against the current working stack.

The existing SOCKS5 configuration should remain documented until the VPN replacement is deployed and verified.

## References

- Prowlarr Settings: https://wiki.servarr.com/prowlarr/settings
- Prowlarr: https://prowlarr.com/
- NordVPN: https://nordvpn.com/
