# Hosting Tuwunel on Ubuntu (Docker Compose)

Tuwunel is a lightweight Matrix homeserver. This guide deploys it using Docker Compose on Ubuntu, with Caddy as a reverse proxy for TLS.

Replace **`yourdomain.com`** throughout with your actual domain.

**Critical:** The `server_name`, namely your domain name, can never be changed later.

## Prerequisites

```bash
# Install Docker (official repository)
# https://docs.docker.com/engine/install/ubuntu/
#
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```



## Setup

### Generate secrets

```bash
# Registration token — share with new members to register
openssl rand -hex 32    # → REGISTRATION_TOKEN

# TURN shared secret — used by both Tuwunel and Coturn
openssl rand -hex 32    # → TURN_SECRET

# LiveKit API key and secret — used by LiveKit and lk-jwt-service
openssl rand -hex 10    # → MRTCKEY
openssl rand -hex 32    # → MRTCSECRET
```

### Create directories

```bash
mkdir -p ~/tuwunel/{data,caddy/etc,caddy/data,caddy/config,coturn,matrix-rtc}
```

`~/tuwunel/data` holds the database and media — back it up regularly.

### Caddyfile

`~/tuwunel/caddy/etc/Caddyfile`:
```
yourdomain.com, yourdomain.com:8448 {
    @wellknown_client path /.well-known/matrix/client
    handle @wellknown_client {
        header Content-Type application/json
        header Access-Control-Allow-Origin *
        respond `{"m.homeserver":{"base_url":"https://yourdomain.com"},"org.matrix.msc4143.rtc_foci":[{"type":"livekit","livekit_service_url":"https://matrix-rtc.yourdomain.com"}]}`
    }
    reverse_proxy tuwunel:6167
}

matrix-rtc.yourdomain.com {
    @jwt_service path /sfu/get* /healthz* /get_token*
    handle @jwt_service {
        reverse_proxy matrix-rtc-jwt:8081
    }
    handle {
        reverse_proxy host.docker.internal:7880 {
            header_up Connection "upgrade"
            header_up Upgrade {http.request.header.Upgrade}
        }
    }
}
```

### Coturn config

`~/tuwunel/coturn/coturn.conf`:
```
use-auth-secret
static-auth-secret=TURN_SECRET
realm=yourdomain.com
min-port=49160
max-port=49200
```

### LiveKit config

`~/tuwunel/matrix-rtc/livekit.yaml` (replace `MRTCKEY` and `MRTCSECRET`):
```yaml
port: 7880
bind_addresses:
  - ""
rtc:
  tcp_port: 7881
  port_range_start: 50100
  port_range_end: 50200
  use_external_ip: true
  enable_loopback_candidate: false
keys:
  MRTCKEY: MRTCSECRET
```

### Docker Compose

`~/tuwunel/docker-compose.yml` (replace all placeholders):

```yaml
networks:
  tuwunel:

services:
  tuwunel:
    image: docker.io/jevolk/tuwunel:latest
    container_name: tuwunel
    restart: always
    networks:
      - tuwunel
    volumes:
      - ./data:/var/lib/tuwunel
    environment:
      TUWUNEL_SERVER_NAME: yourdomain.com
      TUWUNEL_DATABASE_PATH: /var/lib/tuwunel
      TUWUNEL_PORT: "6167"
      TUWUNEL_ADDRESS: 0.0.0.0
      TUWUNEL_MAX_REQUEST_SIZE: "20000000"
      TUWUNEL_ALLOW_REGISTRATION: "true"
      TUWUNEL_REGISTRATION_TOKEN: REGISTRATION_TOKEN
      TUWUNEL_ALLOW_FEDERATION: "true"
      TUWUNEL_TRUSTED_SERVERS: '["matrix.org"]'
      TUWUNEL_TURN_URIS: '["turns:yourdomain.com:5349","turn:yourdomain.com:3478"]'
      TUWUNEL_TURN_SECRET: TURN_SECRET

  caddy:
    image: docker.io/library/caddy:latest
    container_name: caddy
    restart: always
    depends_on:
      - tuwunel
    networks:
      tuwunel:
        aliases:
          - yourdomain.com
          - matrix-rtc.yourdomain.com
    extra_hosts:
      - "host.docker.internal:host-gateway"
    ports:
      - "80:80"
      - "443:443"
      - "8448:8448"
    volumes:
      - ./caddy/etc:/etc/caddy
      - ./caddy/data:/data
      - ./caddy/config:/config

  coturn:
    image: docker.io/coturn/coturn:latest
    container_name: coturn
    restart: always
    network_mode: host
    volumes:
      - ./coturn/coturn.conf:/etc/coturn/turnserver.conf:ro

  matrix-rtc-jwt:
    image: ghcr.io/element-hq/lk-jwt-service:latest
    container_name: matrix-rtc-jwt
    restart: always
    networks:
      - tuwunel
    environment:
      LIVEKIT_JWT_BIND: ":8081"
      LIVEKIT_URL: wss://matrix-rtc.yourdomain.com
      LIVEKIT_KEY: MRTCKEY
      LIVEKIT_SECRET: MRTCSECRET
      LIVEKIT_FULL_ACCESS_HOMESERVERS: yourdomain.com

  matrix-rtc-livekit:
    image: docker.io/livekit/livekit-server:latest
    container_name: matrix-rtc-livekit
    restart: always
    network_mode: host
    command: --config /etc/livekit.yaml
    volumes:
      - ./matrix-rtc/livekit.yaml:/etc/livekit.yaml:ro
```

Caddy handles TLS automatically via Let's Encrypt. It serves `.well-known/matrix/client` directly (intercepting before Tuwunel) and reaches the JWT service by container name on the shared network. LiveKit is reached via `host.docker.internal` since it uses host networking.

The `aliases` on the Caddy network are needed because the Matrix RTC JWT service calls back to the homeserver domain. Network aliases let containers resolve `yourdomain.com` to Caddy within the Docker network.

Reference: https://matrix-construct.github.io/tuwunel/deploying/reverse-proxy-caddy.html

**Prerequisite for Matrix RTC:** Create a DNS record for `matrix-rtc.yourdomain.com` pointing to your server.

**Optional — Coturn as TURN relay for LiveKit** (improves calls on restrictive networks):

Add a second secret to `coturn.conf`:
```
static-auth-secret=YOUR_LIVEKIT_TURN_SECRET
```

Add to the `rtc:` block in `livekit.yaml`:
```yaml
  turn_servers:
    - host: yourdomain.com
      port: 5349
      protocol: tls
      secret: "YOUR_LIVEKIT_TURN_SECRET"
```

Coturn's relay ports (49160–49200) don't conflict with LiveKit's (50100–50200).

## Start

```bash
cd ~/tuwunel
docker compose up -d

# Verify all containers are running
docker compose ps
```

Verification:
```bash
curl https://yourdomain.com/_tuwunel/server_version
curl https://yourdomain.com:8448/_matrix/federation/v1/version
```

You can also use the [Matrix Federation Tester](https://federationtester.matrix.org/).

## Register Accounts

Use Element or any Matrix client pointed at `https://yourdomain.com` and register with the token from the secrets step. **The first registration automatically gets server admin privileges.**

Share with new members: the homeserver URL and registration token.

Once everyone is registered, disable registration by changing the environment variable in `docker-compose.yml`:
```yaml
      TUWUNEL_ALLOW_REGISTRATION: "false"
```
```bash
cd ~/tuwunel
docker compose up -d tuwunel
```

To create accounts directly (useful since Element X doesn't support registration), use the `#admins` room (auto-joined on first login):
```
!admin users create-user <USERNAME> [PASSWORD]
```

## Maintenance

```bash
# View logs
docker compose logs -f tuwunel
docker compose logs -f caddy
docker compose logs -f coturn

# Update all images and restart
cd ~/tuwunel
docker compose pull
docker compose up -d
```

### Backup (Borg + Hetzner Storage Box)

[Hetzner Storage Box](https://www.hetzner.com/storage/storage-box/) provides SSH-accessible storage that works natively with BorgBackup. BX11 (1 TB) is plenty for a small homeserver.

**One-time setup:**

```bash
sudo apt install borgbackup

# Generate a dedicated SSH key
ssh-keygen -t ed25519 -f ~/.ssh/storage-box -N ""

# Upload the key to Hetzner Storage Box
cat ~/.ssh/storage-box.pub | ssh -p 23 uXXXXXX@uXXXXXX.your-storagebox.de install-ssh-key

# Add to SSH config for convenience
cat >> ~/.ssh/config <<EOF

Host storagebox
    HostName uXXXXXX.your-storagebox.de
    User uXXXXXX
    Port 23
    IdentityFile ~/.ssh/storage-box
EOF

# Initialize the Borg repository
borg init --encryption=repokey ssh://storagebox/./tuwunel

# IMPORTANT: back up the repo key — without it, backups are unrecoverable
borg key export ssh://storagebox/./tuwunel ~/borg-key-backup.txt
# Store this key + the BORG_PASSPHRASE somewhere safe (password manager)
```

**Backup script** — `~/tuwunel/backup.sh`:

```bash
#!/bin/bash
set -euo pipefail

export BORG_REPO="ssh://storagebox/./tuwunel"
export BORG_PASSPHRASE="your-passphrase-here"  # or use BORG_PASSCOMMAND with a secret manager

cd ~/tuwunel

docker compose stop tuwunel
borg create --compression zstd \
    "::tuwunel-{now:%Y-%m-%d_%H:%M}" \
    ~/tuwunel/data
docker compose start tuwunel

borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6
borg compact
```

```bash
chmod 700 ~/tuwunel/backup.sh
```

**Automate with cron** (nightly at 03:00):

```bash
(crontab -l 2>/dev/null; echo "0 3 * * * $HOME/tuwunel/backup.sh >> $HOME/tuwunel/backup.log 2>&1") | crontab -
```

**Restore:**

```bash
# List snapshots
borg list ssh://storagebox/./tuwunel

# Restore a specific snapshot
cd ~/tuwunel
docker compose stop tuwunel
mv data data.old
borg extract ssh://storagebox/./tuwunel::tuwunel-2026-03-25_03:00
docker compose start tuwunel
```

### Optional: automatic updates with Watchtower

```bash
docker run -d \
  --name watchtower \
  --restart always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  containrrr/watchtower \
  --cleanup --interval 86400
```

This checks for image updates daily and restarts containers automatically.

## Quick Reference

| Item | Value |
| --- | --- |
| Ports (Caddy) | 80, 443, 8448 |
| Ports (TURN) | 3478, 5349, 49160–49200/udp |
| Ports (LiveKit) | 7881/tcp, 50100–50200/udp |
| Data | `~/tuwunel/data` |
| Compose file | `~/tuwunel/docker-compose.yml` |
| Configs | `~/tuwunel/caddy/etc/Caddyfile`, `~/tuwunel/coturn/coturn.conf`, `~/tuwunel/matrix-rtc/livekit.yaml` |
| Logs | `docker compose logs -f {tuwunel,caddy,coturn,matrix-rtc-jwt,matrix-rtc-livekit}` |
| Admin | `#admins:yourdomain.com` (auto-joined) |
| Clients | Element, FluffyChat, Cinny, SchildiChat |
