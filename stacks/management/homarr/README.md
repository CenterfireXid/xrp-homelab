# Homarr

Homarr is the application dashboard for XRP Homelab. It runs on the Raspberry Pi 5 through Docker Compose and provides a single interface for links, service integrations, Docker status, and selected container lifecycle controls.

## Purpose

Homarr provides the day-to-day landing page for self-hosted applications on XRP Homelab.

The deployment also connects Homarr to Docker so newly deployed containers can be discovered. Docker access is routed through a restricted LinuxServer socket proxy instead of mounting `/var/run/docker.sock` directly into Homarr.

## Deployment

| Item | Value |
| --- | --- |
| Platform | Raspberry Pi 5 (ARM64) |
| Deployment method | Docker Compose |
| Homarr image | `ghcr.io/homarr-labs/homarr:v1.77.2` |
| Socket proxy image | `lscr.io/linuxserver/socket-proxy:3.4.4-r0-ls97` |
| Compose project | `homarr` |
| Containers | `homarr`, `homarr-socket-proxy` |
| Homarr runtime UID/GID | `1000:1000` |
| Database | SQLite |
| Host port | `7575/tcp` |

This document describes the verified running `v1.77.2` deployment. Upgrade, backup, and restore procedures are intentionally deferred to a later documentation revision as during the deployment and testing phase, V2 was introduced which has fairly significant database changes.

## Architecture

```text
Browser
  |
  | TCP 7575
  v
Raspberry Pi 5
  |
  +-- homarr
  |     |
  |     +-- /appdata
  |     |     |
  |     |     v
  |     |  /srv/docker/homarr
  |     |
  |     +-- docker-api network
  |             |
  |             v
  |      homarr-socket-proxy
  |             |
  |             | read-only socket mount
  |             v
  |      /var/run/docker.sock
  |
  +-- Docker Engine
```

Homarr is attached to the normal Compose network for its web interface and outbound application integrations. It is also attached to the private `docker-api` network so it can reach `homarr-socket-proxy`.

The proxy is attached only to `docker-api`. TCP `2375` is not published on the host or LAN. This decision was made to restrict access and prevent homarr from creating/removing containers, mounting host paths, etc. thereby reducing the attack surface for potential miscreants.

## Directory Structure

Live deployment definition:

```text
/opt/stacks/homarr/
├── compose.yaml
└── .env
```

Persistent application data:

```text
/srv/docker/homarr/
├── db/
├── redis/
└── trusted-certificates/
```

The live `.env` contains the Homarr encryption key and is not stored in the public repository.

Repository copy:

```text
stacks/
└── management/
    └── homarr/
        ├── compose.yaml
        ├── README.md
        └── .env.example
```

## Docker Compose

Local deployment definition:

```text
/opt/stacks/homarr/compose.yaml
```

Sanitized deployed configuration:

```yaml
services:
  homarr:
    image: ghcr.io/homarr-labs/homarr:v1.77.2
    container_name: homarr
    restart: unless-stopped
    depends_on:
      - homarr-socket-proxy
    environment:
      SECRET_ENCRYPTION_KEY: ${SECRET_ENCRYPTION_KEY}
      PUID: "1000"
      PGID: "1000"
      DOCKER_HOSTNAMES: homarr-socket-proxy
      DOCKER_PORTS: "2375"
    ports:
      - "7575:7575"
    volumes:
      - /srv/docker/homarr:/appdata
    networks:
      - default
      - docker-api

  homarr-socket-proxy:
    image: lscr.io/linuxserver/socket-proxy:3.4.4-r0-ls97
    container_name: homarr-socket-proxy
    restart: unless-stopped
    environment:
      CONTAINERS: "1"
      EVENTS: "1"
      PING: "1"
      VERSION: "1"
      ALLOW_LOGS: "1"
      ALLOW_START: "1"
      ALLOW_STOP: "1"
      ALLOW_RESTARTS: "1"
      POST: "0"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    read_only: true
    tmpfs:
      - /run
    networks:
      - docker-api

networks:
  docker-api:
    internal: true
```

## Ports

| Host Port | Container Port | Purpose |
| ---: | ---: | --- |
| `7575/tcp` | `7575/tcp` | Homarr web interface |

The socket proxy listens on TCP `2375` only inside the private `docker-api` network. No host port is published for the proxy.

## Persistent Storage

| Host Path | Container Path | Contents |
| --- | --- | --- |
| `/srv/docker/homarr` | `/appdata` | Homarr database, Redis state, trusted certificates, boards, application configuration, integrations, and other persistent state |

The persistent directory is owned by UID/GID `1000:1000`, matching the Homarr `PUID` and `PGID` configuration.

## Networking

Homarr uses two Compose networks:

- `homarr_default` provides normal container networking and access to the published web interface.
- `homarr_docker-api` is an internal network shared only by Homarr and the socket proxy.

The socket proxy is not published to the host. Homarr reaches it by Compose service name:

```text
homarr-socket-proxy:2375
```

## Configuration

| Setting | Value | Reason |
| --- | --- | --- |
| Homarr image | `ghcr.io/homarr-labs/homarr:v1.77.2` | Pins the verified running Homarr release |
| Socket proxy image | `lscr.io/linuxserver/socket-proxy:3.4.4-r0-ls97` | Pins the verified proxy build |
| `PUID` / `PGID` | `1000` / `1000` | Matches ownership of `/srv/docker/homarr` |
| `SECRET_ENCRYPTION_KEY` | Local `.env` | Keeps the encryption key out of GitHub |
| `DOCKER_HOSTNAMES` | `homarr-socket-proxy` | Sends Homarr Docker API traffic through the proxy |
| `DOCKER_PORTS` | `2375` | Internal proxy listener |
| `CONTAINERS` | `1` | Allows container inventory |
| `ALLOW_LOGS` | `1` | Allows container log access through the proxy |
| `ALLOW_START` | `1` | Allows Homarr to start containers |
| `ALLOW_STOP` | `1` | Allows Homarr to stop containers |
| `ALLOW_RESTARTS` | `1` | Allows Homarr to restart containers |
| `POST` | `0` | Blocks unrestricted Docker POST operations |

The proxy intentionally does not grant general Docker API write access. Container removal is not available through this configuration.

Application links, boards, users, and application-specific integrations are configured in the Homarr UI and persist under `/appdata`. Integration credentials are not stored in this repository.

## Initial Deployment

This procedure assumes Docker Engine and Docker Compose are already installed on the XRP Homelab host.

### 1. Check architecture and conflicts

```bash
uname -m

docker compose version
docker compose ls

sudo ss -ltnp | grep ':7575 ' || true
docker ps -a --filter 'name=homarr'

sudo ls -ld \
  /opt/stacks/homarr \
  /srv/docker/homarr \
  2>/dev/null || true
```

The host should report:

```text
aarch64
```

Port `7575` should be free before the first deployment.

### 2. Create deployment and persistent-data directories

```bash
sudo mkdir -p /opt/stacks/homarr
sudo mkdir -p /srv/docker/homarr

sudo chown -R 1000:1000 /srv/docker/homarr

ls -ld /opt/stacks/homarr /srv/docker/homarr
```

Do not use `chmod 777` as a permissions workaround.

### 3. Install the Compose definition

Place the repository's `compose.yaml` at:

```text
/opt/stacks/homarr/compose.yaml
```

For example, from a local repository checkout:

```bash
sudo cp stacks/management/homarr/compose.yaml \
  /opt/stacks/homarr/compose.yaml
```

### 4. Create the local encryption key

Generate the Homarr encryption key and store it in the live `.env` file:

```bash
sudo sh -c \
  'umask 077; printf "SECRET_ENCRYPTION_KEY=%s\n" "$(openssl rand -hex 32)" > /opt/stacks/homarr/.env'

sudo chown "$USER":"$USER" /opt/stacks/homarr/.env
sudo chmod 600 /opt/stacks/homarr/.env
```

Verify the key length without printing the key:

```bash
awk -F= \
  '/^SECRET_ENCRYPTION_KEY=/{print "Encryption key length:", length($2)}' \
  /opt/stacks/homarr/.env

ls -l /opt/stacks/homarr/.env
```

Expected key length:

```text
64
```

### 5. Validate

```bash
cd /opt/stacks/homarr

docker compose config --quiet

docker compose config \
  | sed -E 's/(SECRET_ENCRYPTION_KEY:).*/\1 <redacted>/'
```

Confirm the resolved configuration contains:

- `ghcr.io/homarr-labs/homarr:v1.77.2`
- `lscr.io/linuxserver/socket-proxy:3.4.4-r0-ls97`
- host port `7575`
- `/srv/docker/homarr:/appdata`
- `PUID=1000` and `PGID=1000`
- `DOCKER_HOSTNAMES=homarr-socket-proxy`
- `DOCKER_PORTS=2375`
- `POST=0`
- the internal `docker-api` network
- no published host port for the socket proxy

Resolve validation errors before continuing.

### 6. Pull the images

```bash
docker compose pull
```

Both images must pull successfully on the ARM64 host before deployment continues.

### 7. Deploy

```bash
docker compose up -d
docker compose ps
docker compose logs --tail=100
```

Expected containers:

```text
homarr
homarr-socket-proxy
```

Only `homarr` should publish a host port.

### 8. Verify the web interface and persistent data

Check the local endpoint:

```bash
curl -I http://127.0.0.1:7575
```

A fresh installation may return a redirect to `/init`.

Inspect the persistent directory:

```bash
ls -lah /srv/docker/homarr
```

After first startup, Homarr creates persistent state below `/appdata`, including the database and Redis directories.

### 9. Verify Docker discovery through the proxy

Shoutout to ChatGPT, helped me figure out the proxy settings and implementation to keep my homarr instance more secure.

Test the proxy endpoint from inside Homarr:

```bash
docker exec homarr node -e \
"fetch('http://homarr-socket-proxy:2375/_ping')
 .then(async r => console.log('Proxy ping:', r.status, await r.text()))
 .catch(e => { console.error(e); process.exit(1) })"
```

Expected:

```text
Proxy ping: 200 OK
```

Verify container inventory:

```bash
docker exec homarr node -e \
"fetch('http://homarr-socket-proxy:2375/containers/json?all=1')
 .then(async r => {
   console.log('Containers API:', r.status);
   const c = await r.json();
   console.log(c.map(x => x.Names[0]).join('\n'));
 })
 .catch(e => { console.error(e); process.exit(1) })"
```

Expected:

```text
Containers API: 200
```

The output should include the Docker containers currently running on the host.

### 10. Complete first-run setup

Open:

```text
http://<PI-IP>:7575
```

Complete the Homarr initialization flow and create the administrator account.

Docker-discovered services can be added to Homarr from its Docker tools. Containers deployed later should appear in Docker discovery automatically; creating an application entry or dashboard tile remains an explicit Homarr action.

Application-specific credentials should use dedicated accounts or API keys where available. Do not store those credentials in the repository.

### 11. Verify in Portainer

Confirm Portainer automatically detects:

- Compose project `homarr`
- container `homarr`
- container `homarr-socket-proxy`
- published port `7575/tcp` on Homarr
- `/srv/docker/homarr:/appdata`
- internal network `homarr_docker-api`

No separate Portainer deployment is required. Do not recreate the containers through Portainer's Add Container workflow.

## Access

Local web interface:

```text
http://<PI-IP>:7575
```

The current Compose definition does not publish Homarr directly to the public Internet.

## Portainer

Docker Compose owns the Homarr deployment definition. Portainer reads the resulting Docker state and is used for status, logs, resource use, networks, mounts, and routine troubleshooting.

Permanent configuration changes belong in `/opt/stacks/homarr/compose.yaml`, not in Portainer's Add Container workflow.

## Security Considerations

Homarr does not mount `/var/run/docker.sock` directly.

The `homarr-socket-proxy` container does mount the Docker socket read-only so it can proxy allowed API requests. Access to that proxy is reduced in three ways:

- TCP `2375` is not published on the host.
- The proxy is attached only to the internal `docker-api` network.
- General Docker `POST` access is disabled with `POST=0`.

The proxy explicitly permits container inventory, logs, start, stop, and restart operations. This is still privileged access to Docker functionality and should not be exposed to untrusted containers or networks.

The real `SECRET_ENCRYPTION_KEY` remains in `/opt/stacks/homarr/.env` and must not be committed.

Homarr is exposed over plain HTTP on the local network. Public Internet exposure is not part of this deployment.

Application integrations should use dedicated API keys or least-privilege accounts where supported.

## Troubleshooting

### Compose cannot read `.env`

**Symptom**

`docker compose config` reports:

```text
open /opt/stacks/homarr/.env: permission denied
```

**Cause**

The `.env` file is not readable by the account running Docker Compose.

**Resolution**

Keep the file private, but make the Compose operator its owner:

```bash
sudo chown "$USER":"$USER" /opt/stacks/homarr/.env
sudo chmod 600 /opt/stacks/homarr/.env
```

Then retry:

```bash
cd /opt/stacks/homarr
docker compose config --quiet
```

### Homarr does not discover Docker containers

**Symptom**

The Homarr Docker page is empty or reports an endpoint error.

**Resolution**

Check both containers:

```bash
cd /opt/stacks/homarr
docker compose ps
```

Then test the proxy from Homarr:

```bash
docker exec homarr node -e \
"fetch('http://homarr-socket-proxy:2375/_ping')
 .then(async r => console.log(r.status, await r.text()))
 .catch(e => { console.error(e); process.exit(1) })"
```

A working endpoint returns `200 OK`.

Review proxy logs if the request is blocked:

```bash
docker compose logs --tail=100 homarr-socket-proxy
```

### Docker action is denied

**Symptom**

Homarr can see a container but a Docker action fails.

**Cause**

The socket proxy intentionally allows only selected Docker API operations.

**Resolution**

Check whether the requested action is supposed to be permitted by the current proxy policy. Do not enable unrestricted `POST=1` only to make an unrelated feature work.

The current deployment intentionally permits start, stop, and restart but not container removal.

### Redis memory overcommit warning

**Symptom**

Homarr startup logs include a Redis warning that `vm.overcommit_memory` is disabled.

**Current state**

The warning was observed during the initial deployment while the host reported:

```text
vm.overcommit_memory = 0
```

Homarr completed startup and served the web interface normally. No host-wide sysctl change was made as part of this deployment.

## References

- Homarr Docker installation: https://homarr.dev/docs/getting-started/installation/docker/
- Homarr environment variables: https://homarr.dev/docs/advanced/environment-variables/
- Homarr Docker integration: https://homarr.dev/docs/integrations/docker/
- Homarr releases: https://github.com/homarr-labs/homarr/releases
- LinuxServer socket-proxy documentation: https://docs.linuxserver.io/images/docker-socket-proxy/
- LinuxServer socket-proxy source: https://github.com/linuxserver/docker-socket-proxy
