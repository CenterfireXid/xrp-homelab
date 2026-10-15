# qBittorrent

qBittorrent is a headless BitTorrent client deployed on XRP Homelab through Docker Compose. It provides the download client for the media-management environment and currently stores download data on my NAS.

> [!WARNING]
> **Legal notice:** qBittorrent is documented here for educational and lawful file-transfer use only. Do not use this deployment to download or distribute copyrighted material without authorization.

## Purpose

qBittorrent provides a centrally managed download client with a browser-based WebUI. Application configuration is stored locally on the Raspberry Pi, while downloaded data is written directly to my NAS media share.

Downloads use separate incomplete and complete directories:

```text
/downloads/incomplete
        ↓
  download completes
        ↓
/downloads/complete
```

The deployment uses an authenticated NordVPN SOCKS5 proxy for BitTorrent traffic. Proxy credentials are configured inside qBittorrent rather than the compose file.

## Deployment

- **Application:** qBittorrent
- **Application version:** 5.2.3
- **Container image:** `qbittorrentofficial/qbittorrent-nox:5.2.3-1`
- **Compose project:** `qbittorrent`
- **Container:** `qbittorrent`
- **Architecture:** ARM64 (`aarch64`)
- **Host Compose path:** `/opt/stacks/qbittorrent/compose.yaml`
- **Persistent configuration:** `/srv/docker/qbittorrent/config`
- **Download storage:** `/mnt/unas-media/downloads`
- **Restart policy:** `unless-stopped`
- **Compose network:** default project network (`qbittorrent_default`)

## Architecture

qBittorrent runs as a standalone Compose project on the XRP Homelab Raspberry Pi.

Application state remains on local storage under `/srv/docker/qbittorrent/config`. Download data is written to the UNAS NFS-backed media filesystem through `/mnt/unas-media/downloads`.

Only the download subtree is exposed to qBittorrent. The container does not receive access to the complete media library.

Internet-facing BitTorrent traffic is configured to use an authenticated NordVPN SOCKS5 proxy. Other media services can integrate with qBittorrent through its Web API without being placed behind the same proxy.

## Directory Structure

Host paths:

```text
/opt/stacks/qbittorrent/
└── compose.yaml

/srv/docker/qbittorrent/
└── config/

/mnt/unas-media/downloads/
├── complete/
└── incomplete/
```

Container paths:

```text
/config
/downloads
/downloads/complete
/downloads/incomplete
```

## Docker Compose

The repository copy of `compose.yaml` is the source of truth for the permanent container definition.

The deployment uses the official qBittorrent no-X image pinned to `5.2.3-1`. The top-level Compose `version` field is intentionally omitted.

The container receives supplementary group ID `988` through `PAGID` so that the qBittorrent process can access the NFS-backed download directories.

## Ports

| Host | Container | Protocol | Purpose |
| --- | --- | --- | --- |
| `10095` | `10095` | TCP | qBittorrent WebUI |
| `51413` | `51413` | TCP | BitTorrent listening port |
| `51413` | `51413` | UDP | BitTorrent listening port |

Port `10095` is the WebUI access port for the local network.

## Persistent Storage

### Application state

```text
/srv/docker/qbittorrent/config:/config
```

This directory contains qBittorrent configuration, WebUI settings, categories, logs, proxy configuration, and torrent session state.

The proxy credentials stored in the qBittorrent configuration are sensitive. Backups of this directory must be protected accordingly.

### Download data

```text
/mnt/unas-media/downloads:/downloads
```

The NAS NFS share provides the persistent download filesystem.

qBittorrent uses:

```text
/downloads/incomplete
/downloads/complete
```

A functional storage test confirmed that the container can create files in the incomplete directory, move them into the complete directory, and delete them. The underlying NFS export also supports hardlinks within the media filesystem.

Download data is protected as part of the UNAS data strategy rather than the qBittorrent application-state backup.

## Networking

The deployment uses the default Compose network.

qBittorrent publishes its WebUI and BitTorrent listening ports through the host. No host networking, privileged mode, Docker socket access, host devices, or elevated capabilities are required.

### SOCKS5 proxy

The current deployment uses an authenticated NordVPN SOCKS5 proxy:

```text
dallas.us.socks.nordhold.net:1080
```

The following qBittorrent proxy settings are enabled:

- SOCKS5 authentication
- hostname lookup through the proxy
- proxy use for BitTorrent purposes
- proxy use for peer connections

<img width="731" height="474" alt="image" src="https://github.com/user-attachments/assets/41358ff7-96b7-4ab3-aa11-d779f724f511" />

Torrent-address testing confirmed that BitTorrent traffic presented NordVPN proxy addresses rather than the residential public IP. A fail-closed test was also performed by temporarily configuring an unreachable proxy port. During that test no BitTorrent address was observed rather than traffic falling back to the residential connection.

SOCKS5 is an application proxy, not a full VPN tunnel. A separate full-tunnel design may be evaluated in the future but is not part of this deployment.

### UDP tracker behavior through SOCKS5

The current deployment has a known limitation with UDP trackers.

HTTP and HTTPS trackers have been observed working through the configured SOCKS5 proxy, while UDP trackers may report `Permission denied`. qBittorrent has also logged repeated SOCKS5 `End of file` errors during these failures.

Changing between tested NordVPN SOCKS5 endpoints did not resolve the behavior. The proxy configuration is retained because its privacy behavior was verified, with the understanding that some peer-discovery paths may not function through the current SOCKS5 setup.

Do not disable the proxy merely to restore UDP tracker connectivity. A full VPN tunnel may be evaluated separately in the future.

## Configuration

### Download paths

Configure qBittorrent to use:

```text
Default Save Path: /downloads/complete
Incomplete Torrent Path: /downloads/incomplete
Use another path for incomplete torrents: enabled
```

The persisted qBittorrent settings correspond to:

```text
Session\DefaultSavePath=/downloads/complete
Session\TempPath=/downloads/incomplete
Session\TempPathEnabled=true
```

### SOCKS5 settings

Under **Tools → Options → Connection**, configure the authenticated SOCKS5 proxy using the operator's NordVPN service credentials.

Use:

```text
Type: SOCKS5
Host: dallas.us.socks.nordhold.net
Port: 1080
Authentication: enabled
Perform hostname lookup via proxy: enabled
Use proxy for BitTorrent purposes: enabled
Use proxy for peer connections: enabled
```

### Advanced network binding

The verified Advanced settings are:

```text
Network interface: Any interface
Optional IP address to bind to: All addresses
```

No special interface binding is currently required for the SOCKS5 deployment.

### Excluded file types

qBittorrent is configured to exclude executable, script, package, and disk-image formats that are not expected in this media workflow.

The exclusion list is a defense-in-depth measure. It does not perform malware scanning or validate the actual contents of a file.

Current exclusions:

```text
*.exe
*.com
*.bat
*.cmd
*.msi
*.msp
*.msix
*.scr
*.pif
*.cpl
*.jar
*.js
*.jse
*.vbs
*.vbe
*.wsf
*.wsh
*.ps1
*.psm1
*.psd1
*.hta
*.reg
*.lnk
*.scf
*.sh
*.bash
*.zsh
*.fish
*.apk
*.appx
*.appxbundle
*.deb
*.rpm
*.dmg
*.pkg
*.iso
*.img
```

Archive formats such as `.zip`, `.rar`, and `.7z` are not globally excluded because they may contain legitimate authorized content. Common media, subtitle, metadata, and ebook formats are also left available.

## Initial Deployment

Create the stack and persistent configuration directories:

```bash
sudo mkdir -p /opt/stacks/qbittorrent
sudo mkdir -p /srv/docker/qbittorrent/config
```

Confirm that the UNAS media mount and required download directories are available:

```bash
findmnt /mnt/unas-media
ls -ld \
  /mnt/unas-media/downloads \
  /mnt/unas-media/downloads/complete \
  /mnt/unas-media/downloads/incomplete
```

Copy the repository `compose.yaml` to:

```text
/opt/stacks/qbittorrent/compose.yaml
```

Validate the resolved Compose configuration before starting the service:

```bash
cd /opt/stacks/qbittorrent
docker compose config
docker compose config --images
```

Review the image, ports, bind mounts, environment values, network, and restart policy.

Pull the image:

```bash
docker compose pull
```

A successful pull is also a practical confirmation that the pinned image is available for the host architecture.

Deploy qBittorrent:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Open the WebUI and complete the application configuration described above. Replace any initial WebUI credentials with your own credentials.

Verify storage access from the application before relying on the deployment. Confirm that qBittorrent can write to `/downloads/incomplete` and move completed data to `/downloads/complete`.

Configure the SOCKS5 proxy and perform a privacy test using lawful test data. Confirm that the observed BitTorrent address belongs to the proxy path rather than the residential Internet connection.

A proxy-failure test should also confirm that qBittorrent does not fall back to the residential connection when the configured proxy is unreachable.

## Access

The WebUI is available on the XRP Homelab host at:

```text
http://<pi-ip>:10095
```

Authentication is required.

Do not expose the WebUI directly to the public Internet.

## Portainer

The Compose deployment appears automatically in Portainer after it is started from the CLI.

Use Portainer to inspect:

- container state
- logs
- resource use
- network membership
- bind mounts
- console access
- restart behavior

Do not recreate or redefine the qBittorrent container through Portainer's **Add Container** interface. Docker Compose remains the permanent deployment definition.

## Backup

Back up:

```text
/srv/docker/qbittorrent/config/
```

This includes application configuration, WebUI settings, proxy configuration, categories, and torrent session state.

> [!IMPORTANT]
> Treat this backup as **sensitive** because qBittorrent stores the configured SOCKS5 credentials in its persistent configuration.

The following download directories are not part of the qBittorrent application-state backup:

```text
/mnt/unas-media/downloads/complete
/mnt/unas-media/downloads/incomplete
```

They belong to the separate NAS storage and it's own data-protection strategy.

Before major qBittorrent upgrades, take a fresh backup of the configuration directory.

## Restore

On a clean XRP Homelab host:

1. Restore or configure the UNAS NFS mount at `/mnt/unas-media`.
2. Confirm `/mnt/unas-media/downloads`, `complete`, and `incomplete` are available.
3. Create `/opt/stacks/qbittorrent`.
4. Create `/srv/docker/qbittorrent/config`.
5. Restore the qBittorrent configuration backup into `/srv/docker/qbittorrent/config`.
6. Restore the repository `compose.yaml` to `/opt/stacks/qbittorrent/compose.yaml`.
7. Validate, pull, and deploy the service.

Commands:

```bash
cd /opt/stacks/qbittorrent
docker compose config
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

After restoration, verify:

- WebUI authentication
- download paths
- NFS write access
- proxy settings
- proxy-visible BitTorrent address
- absence of residential-IP fallback
- expected application state

Do not assume a running container proves that storage or proxy behavior has been restored correctly.

## Update Procedure

The container image is pinned. Do not replace the tag with `latest`.

Before updating:

1. Review the qBittorrent and container-image release notes.
2. Check for configuration or migration requirements.
3. Back up `/srv/docker/qbittorrent/config`.
4. Update the pinned image tag in the repository and host Compose file.

Validate and pull the new image:

```bash
cd /opt/stacks/qbittorrent
docker compose config
docker compose pull
```

Redeploy:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Verify the application version:

```bash
docker exec qbittorrent qbittorrent-nox --version
```

After an update, verify WebUI access, NFS storage behavior, proxy configuration, and the proxy-visible BitTorrent address.

## Security Considerations

- Treat `/srv/docker/qbittorrent/config` and its backups as sensitive because the proxy password is stored in the application configuration.
- Do not expose the WebUI directly to the public Internet.
- Keep proxy hostname lookup enabled to avoid intentionally moving application hostname resolution outside the configured proxy path.
- Keep proxy use enabled for BitTorrent and peer connections.
- Do not disable the proxy as a routine troubleshooting step.
- The file exclusion list is defense in depth and is not a malware scanner.
- qBittorrent receives access only to `/mnt/unas-media/downloads`, not the entire media filesystem.
- Avoid broad permission changes such as `chmod 777` when troubleshooting NFS access.
- A SOCKS5 proxy is not equivalent to a full VPN tunnel.

## Troubleshooting

### `/downloads` ownership warning during startup

**Symptom**

Container logs contain:

```text
chown: /downloads: Operation not permitted
```

**Cause**

`/downloads` is backed by the NAS NFS share. The NFS server does not permit the container startup process to change ownership of the mounted directory.

**Resolution**

No permission change is currently required.

Write, move, and delete operations were tested successfully from inside the container, including moving a test file from `/downloads/incomplete` to `/downloads/complete`.

Do not use `chmod 777` or change UNAS ownership solely to suppress this warning.

### Supplementary group startup warning

The container may log:

```text
addgroup: group 'qbtGroup-988' in use
```

The warning has not prevented the qBittorrent process from accessing the configured NFS download directories in the verified deployment. Troubleshoot it only if accompanied by an actual storage-access failure.

### UDP trackers report `Permission denied`

**Symptom**

Some UDP trackers appear as `Not working` with:

```text
Permission denied
```

qBittorrent may also log:

```text
SOCKS5 proxy error ... Message: "End of file"
```

**Cause**

The behavior has been observed when qBittorrent attempts UDP tracker communication through the current authenticated NordVPN SOCKS5 configuration. HTTP and HTTPS tracker communication has continued to work.

**Resolution**

No production workaround is currently applied.

Do not disable the BitTorrent proxy or peer proxy merely to restore UDP tracker connectivity. The existing configuration is retained because torrent-address and proxy-failure testing confirmed the intended residential-IP protection.

A full VPN tunnel may be evaluated as a separate future deployment.

### Container is running but downloads cannot be written

Do not treat `docker compose ps` showing `Up` as proof that storage is functional.

Check:

```bash
findmnt /mnt/unas-media
ls -ld \
  /mnt/unas-media/downloads \
  /mnt/unas-media/downloads/complete \
  /mnt/unas-media/downloads/incomplete

docker compose logs --tail=100
```

Confirm the NAS NFS mount is active before changing ownership or permissions.

## References

- qBittorrent: https://www.qbittorrent.org/
- qBittorrent GitHub: https://github.com/qbittorrent/qBittorrent
- qBittorrent no-X container: https://github.com/qbittorrent/docker-qbittorrent-nox
- NordVPN qBittorrent proxy setup: https://support.nordvpn.com/hc/en-us/articles/20195967385745-NordVPN-proxy-setup-for-qBittorrent
