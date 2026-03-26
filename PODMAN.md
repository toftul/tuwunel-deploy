# Hosting Tuwunel on AlmaLinux/CentOS/RockyLinux/RHEL (Podman Quadlet)

Tuwunel is a lightweight Matrix homeserver. This guide deploys it as a rootless Podman container managed by systemd via Quadlet, with Caddy as a reverse proxy for TLS.

Replace **`yourdomain.com`** throughout with your actual domain.

**Critical:** The `server_name`, namely your domain name, can never be changed later. 

## Why this tutorial exist

It was very hard to figure it out by reading just the docs, so I put everything together in one spot. 

## Prerequisites

```bash
sudo dnf install -y podman firewalld
sudo systemctl enable --now firewalld
sudo loginctl enable-linger $USER

# Allow rootless containers to bind port 80+
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-unprivileged-ports.conf
sudo sysctl -w net.ipv4.ip_unprivileged_port_start=80

# Open all required ports
# 80/443: Caddy TLS | 8448: Matrix federation | 3478/5349: TURN
# 49160-49200: Coturn relay | 7881+50100-50200: LiveKit (Matrix RTC)
sudo firewall-cmd --permanent --add-service={http,https}
sudo firewall-cmd --permanent --add-port={8448/tcp,3478/tcp,3478/udp,5349/tcp,5349/udp}
sudo firewall-cmd --permanent --add-port={49160-49200/udp,7881/tcp,50100-50200/udp}
sudo firewall-cmd --reload
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
mkdir -p ~/tuwunel-data ~/caddy-etc ~/caddy-data ~/caddy-config ~/coturn-etc ~/matrix-rtc
mkdir -p ~/.config/containers/systemd
```

`~/tuwunel-data` holds the database and media — back it up regularly.

### Podman network

`~/.config/containers/systemd/tuwunel.network` — shared network so containers can reach each other by name:
```ini
[Network]
NetworkName=tuwunel
```

### Tuwunel container

`~/.config/containers/systemd/tuwunel.container`:

```ini
[Container]
Image=docker.io/jevolk/tuwunel:latest
ContainerName=tuwunel
Network=tuwunel.network
AutoUpdate=registry
Volume=/home/ivan/tuwunel-data:/var/lib/tuwunel:Z

Environment=TUWUNEL_SERVER_NAME=yourdomain.com
Environment=TUWUNEL_DATABASE_PATH=/var/lib/tuwunel
Environment=TUWUNEL_PORT=6167
Environment=TUWUNEL_ADDRESS=0.0.0.0
Environment=TUWUNEL_MAX_REQUEST_SIZE=20000000
Environment=TUWUNEL_ALLOW_REGISTRATION=true
Environment=TUWUNEL_REGISTRATION_TOKEN=REGISTRATION_TOKEN
Environment=TUWUNEL_ALLOW_FEDERATION=true
Environment=TUWUNEL_TRUSTED_SERVERS=["matrix.org"]
Environment=TUWUNEL_TURN_URIS=["turns:yourdomain.com:5349","turn:yourdomain.com:3478"]
Environment=TUWUNEL_TURN_SECRET=TURN_SECRET

[Service]
Restart=always

[Install]
WantedBy=default.target
```


### Caddy container

`~/caddy-etc/Caddyfile`:
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
        reverse_proxy host.containers.internal:7880 {
            header_up Connection "upgrade"
            header_up Upgrade {http.request.header.Upgrade}
        }
    }
}
```

`~/.config/containers/systemd/caddy.container`:
```ini
[Unit]
Requires=tuwunel.service
After=tuwunel.service

[Container]
Image=docker.io/library/caddy:latest
ContainerName=caddy
# Domain aliases resolve yourdomain.com inside the container network.
# Needed because rootless Podman lacks hairpin NAT — containers can’t reach
# the host’s public IP. The Matrix RTC JWT service calls back to the
# homeserver domain, so this alias is required.
Network=tuwunel.network:alias=yourdomain.com,alias=matrix-rtc.yourdomain.com
PublishPort=80:80
PublishPort=443:443
PublishPort=8448:8448
AutoUpdate=registry
Volume=/home/ivan/caddy-etc:/etc/caddy:Z
Volume=/home/ivan/caddy-data:/data:Z
Volume=/home/ivan/caddy-config:/config:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Caddy handles TLS automatically via Let’s Encrypt. It serves `.well-known/matrix/client` directly (intercepting before Tuwunel) and reaches the JWT service by container name on the shared network. LiveKit is reached via `host.containers.internal` since it uses host networking.

Reference: https://matrix-construct.github.io/tuwunel/deploying/reverse-proxy-caddy.html

### Coturn container (TURN server)

`~/coturn-etc/coturn.conf`:
```
use-auth-secret
static-auth-secret=TURN_SECRET
realm=yourdomain.com
min-port=49160
max-port=49200
```

`~/.config/containers/systemd/coturn.container`:
```ini
[Container]
Image=docker.io/coturn/coturn:latest
ContainerName=coturn
Network=host
AutoUpdate=registry
Volume=/home/ivan/coturn-etc/coturn.conf:/etc/coturn/turnserver.conf:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

Coturn uses `Network=host` because container runtimes handle large UDP port ranges poorly. The relay range (49160–49200) is sufficient for a small homeserver.

### Matrix RTC containers (Element Call)

Matrix RTC enables voice/video calls via Element Call using two services: `lk-jwt-service` (JWT issuer) and LiveKit (WebRTC SFU).

**Prerequisite:** Create a DNS record for `matrix-rtc.yourdomain.com` pointing to your server.

`~/matrix-rtc/livekit.yaml` (replace `MRTCKEY` and `MRTCSECRET`):
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

`~/.config/containers/systemd/matrix-rtc-jwt.container` (replace `MRTCKEY`, `MRTCSECRET`):
```ini
[Container]
Image=ghcr.io/element-hq/lk-jwt-service:latest
ContainerName=matrix-rtc-jwt
Network=tuwunel.network
AutoUpdate=registry
Environment=LIVEKIT_JWT_BIND=:8081
Environment=LIVEKIT_URL=wss://matrix-rtc.yourdomain.com
Environment=LIVEKIT_KEY=MRTCKEY
Environment=LIVEKIT_SECRET=MRTCSECRET
Environment=LIVEKIT_FULL_ACCESS_HOMESERVERS=yourdomain.com

[Service]
Restart=always

[Install]
WantedBy=default.target
```

`~/.config/containers/systemd/matrix-rtc-livekit.container`:
```ini
[Container]
Image=docker.io/livekit/livekit-server:latest
ContainerName=matrix-rtc-livekit
Network=host
AutoUpdate=registry
Exec=--config /etc/livekit.yaml
Volume=/home/ivan/matrix-rtc/livekit.yaml:/etc/livekit.yaml:Z

[Service]
Restart=always

[Install]
WantedBy=default.target
```

LiveKit uses host networking for reliable WebRTC (large UDP port range). The JWT service joins the shared `tuwunel` network so Caddy can reach it by name.

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

Coturn’s relay ports (49160–49200) don’t conflict with LiveKit’s (50100–50200).

## Start

```bash
systemctl --user daemon-reload
systemctl --user enable --now podman-auto-update.timer

# Caddy auto-starts Tuwunel via Requires= dependency
systemctl --user start coturn caddy matrix-rtc-jwt matrix-rtc-livekit

# Verify
systemctl --user status tuwunel caddy coturn matrix-rtc-jwt matrix-rtc-livekit
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

Once everyone is registered, disable registration:
```
Environment=TUWUNEL_ALLOW_REGISTRATION=false
```
```bash
systemctl --user daemon-reload && systemctl --user restart tuwunel
```

To create accounts directly (useful since Element X doesn’t support registration), use the `#admins` room (auto-joined on first login):
```
!admin users create-user <USERNAME> [PASSWORD]
```

## Maintenance

```bash
# View logs
journalctl --user -u tuwunel -f
journalctl --user -u caddy -f
journalctl --user -u coturn -f

# Manual image update (autoupdates handle this normally)
podman pull docker.io/jevolk/tuwunel:latest
podman pull docker.io/library/caddy:latest
systemctl --user restart tuwunel caddy

# Backup (stop first for consistency)
systemctl --user stop tuwunel
tar czf tuwunel-backup-$(date +%F).tar.gz ~/tuwunel-data
systemctl --user start tuwunel
```

## Quick Reference

| Item | Value |
| --- | --- |
| Ports (Caddy) | 80, 443, 8448 |
| Ports (TURN) | 3478, 5349, 49160–49200/udp |
| Ports (LiveKit) | 7881/tcp, 50100–50200/udp |
| Data | `~/tuwunel-data` |
| Quadlet files | `~/.config/containers/systemd/` |
| Configs | `~/caddy-etc/Caddyfile`, `~/coturn-etc/coturn.conf`, `~/matrix-rtc/livekit.yaml` |
| Logs | `journalctl --user -u {tuwunel,caddy,coturn,matrix-rtc-jwt,matrix-rtc-livekit}` |
| Admin | `#admins:yourdomain.com` (auto-joined) |
| Clients | Check https://matrix.org/ecosystem/clients/ |
