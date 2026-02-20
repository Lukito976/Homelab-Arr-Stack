# Homelab: Private Arr Stack (qBittorrent + Gluetun + tsbridge)

This repository documents how to deploy a private, VPN-protected Arr stack using Docker with:

- **[qBittorrent](https://github.com/linuxserver/docker-qbittorrent)** for torrent downloading
- **[Gluetun](https://github.com/qdm12/gluetun)** to route all torrent traffic through NordVPN (OpenVPN) with a kill switch
- **[Prowlarr](https://github.com/linuxserver/docker-prowlarr)** for indexer management
- **[Radarr](https://github.com/linuxserver/docker-radarr)** for movie automation
- **[Sonarr](https://github.com/linuxserver/docker-sonarr)** for TV automation
- **[Jellyseerr](https://hub.docker.com/r/fallenbagel/jellyseerr/)** for media requests
- **[FlareSolverr](https://github.com/FlareSolverr/FlareSolverr)** for bypassing Cloudflare on indexers
- **[tsbridge](https://github.com/jtdowney/tsbridge)** to securely expose services over Tailscale
- No public ports and no LAN/WAN exposure

All external access is restricted to devices on your Tailnet.

---

## Prerequisites

- Linux server (Tested on [Ubuntu 24.04.3 LTS](https://ubuntu.com/download/server))
- [Docker](https://docs.docker.com/engine/install/ubuntu/) + Docker Compose installed
- A [Tailscale](https://tailscale.com/) account
- A Tailscale OAuth client with tag permissions
- A [NordVPN](https://nordvpn.com) subscription with service credentials enabled
- This guide assumes you have either configured Jellyfin, or have followed my [Homelab Program Setup](https://github.com/Lukito976/Homelab-Program-Setup-via-tsbridge)

---

## Directory Layout

### Configuration files

```
~/arr-stack
├── docker-compose.yml
├── tsbridge-arr.toml
└── tsbridge-qbit.toml
```

### Media and application data

**All paths must be on the same filesystem to allow hardlinking.**

```
/media/media-library
├── appdata
│   ├── qbittorrent
│   ├── prowlarr
│   ├── sonarr
│   ├── radarr
│   └── jellyseerr
├── downloads
│   └── torrents
│       ├── movies
│       └── tv
└── media
    ├── movies
    └── tv
```

---

## Step 1 — Create the Arr Stack Directory

```bash
mkdir -p ~/arr-stack
cd ~/arr-stack
```

---

## Step 2 — Configure Tailscale ACLs

Because tsbridge uses OAuth with tags, your Tailnet must allow the tag. If you followed the [Homelab Program Setup](https://github.com/Lukito976/Homelab-Program-Setup-via-tsbridge) guide, you can reuse the same OAuth credentials.

In **Tailscale Admin → Access Controls**, ensure:

```json
{
  "tagOwners": {
    "tag:tsbridge": ["YOUR_TAILSCALE_LOGIN_EMAIL"]
  }
}
```

To generate an OAuth client, go to **Tailscale Admin → Settings → OAuth clients → Generate OAuth client**. Give it **Devices: Read & Write** scope and assign the `tag:tsbridge` tag. Save the Client ID and Client Secret.

---

## Step 3 — Create tsbridge Configuration Files

Create `tsbridge-arr.toml` (used for Arr apps on the Docker bridge network):

```bash
nano ~/arr-stack/tsbridge-arr.toml
```

```toml
[tailscale]
oauth_client_id     = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir           = "/var/lib/tsbridge"
default_tags        = ["tag:tsbridge"]

[[services]]
name         = "prowlarr"
backend_addr = "http://prowlarr:9696"

[[services]]
name         = "sonarr"
backend_addr = "http://sonarr:8989"

[[services]]
name         = "radarr"
backend_addr = "http://radarr:7878"

[[services]]
name         = "jellyseerr"
backend_addr = "http://jellyseerr:5055"
```

Create `tsbridge-qbit.toml` (used for qBittorrent via gluetun's localhost):

```bash
nano ~/arr-stack/tsbridge-qbit.toml
```

```toml
[tailscale]
oauth_client_id     = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir           = "/var/lib/tsbridge"
default_tags        = ["tag:tsbridge"]

[[services]]
name         = "qbittorrent"
backend_addr = "http://127.0.0.1:8081"
```

> **Note:** tsbridge-qbit uses `127.0.0.1` because it runs in the same network namespace as gluetun and qBittorrent via `network_mode: "service:gluetun"`.

---

## Step 4 — Get NordVPN Service Credentials

> ⚠️ These are **not** your NordVPN website login. They are separate credentials used for manual/OpenVPN connections.

1. Go to **my.nordaccount.com**
2. Click **NordVPN** in the left sidebar
3. Scroll to **Manual setup → Service credentials**
4. Copy the username and password — you cannot view them again after leaving the page

If you ever need to regenerate them, click **Regenerate**. You will need to update `docker-compose.yml` and restart gluetun afterward.

---

## Step 5 — Create `docker-compose.yml`

```bash
nano ~/arr-stack/docker-compose.yml
```

```yaml
services:
  tsbridge-arr:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge-arr
    command: ["-config", "/config/tsbridge.toml"]
    cap_add:
      - NET_ADMIN     # Required: Tailscale needs to manage network interfaces
      - SYS_MODULE    # Required: load kernel modules if needed
    devices:
      - /dev/net/tun:/dev/net/tun   # Required: Tailscale TUN device
    volumes:
      - tsbridge-arr-state:/var/lib/tsbridge
      - ./tsbridge-arr.toml:/config/tsbridge.toml:ro
    restart: unless-stopped

  tsbridge-qbit:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge-qbit
    network_mode: "service:gluetun"   # Share gluetun's network namespace
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    devices:
      - /dev/net/tun:/dev/net/tun
    command: ["-config", "/config/tsbridge.toml"]
    volumes:
      - tsbridge-qbit-state:/var/lib/tsbridge
      - ./tsbridge-qbit.toml:/config/tsbridge.toml:ro
    restart: unless-stopped
    depends_on:
      - gluetun

  gluetun:
    image: ghcr.io/qdm12/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - VPN_TYPE=openvpn
      - OPENVPN_USER=YOUR_NORD_SERVICE_USERNAME
      - OPENVPN_PASSWORD=YOUR_NORD_SERVICE_PASSWORD
      - SERVER_COUNTRIES=United States
      - TZ=Etc/UTC
    ports:
      # Port 8081 used because 8080 is taken by Nextcloud AIO
      - "127.0.0.1:8081:8081/tcp"
    healthcheck:
      test: /gluetun-entrypoint healthcheck
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    depends_on:
      gluetun:
        condition: service_healthy   # Wait for VPN to be confirmed up before starting
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8081
    volumes:
      - /media/media-library/appdata/qbittorrent:/config
      - /media/media-library:/data
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/media-library/appdata/prowlarr:/config
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/media-library/appdata/radarr:/config
      - /media/media-library:/data
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/media-library/appdata/sonarr:/config
      - /media/media-library:/data
    restart: unless-stopped

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - TZ=Etc/UTC
    volumes:
      - /media/media-library/appdata/jellyseerr:/app/config
    restart: unless-stopped

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    restart: unless-stopped
    ports:
      - "8191:8191"
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none

volumes:
  tsbridge-arr-state:
  tsbridge-qbit-state:
```

---

## Step 6 — Start the Stack

```bash
cd ~/arr-stack
docker compose up -d
```

Verify qBittorrent is not LAN-exposed (should only show `127.0.0.1`):

```bash
sudo ss -tulpn | grep 8081
```

Verify the VPN is working — these two IPs should be different:

```bash
curl -s https://ifconfig.me/ip && echo
docker exec -it gluetun wget -qO- https://ifconfig.me/ip && echo
```

If gluetun is unhealthy after startup, check its logs:

```bash
docker logs gluetun --tail 20
```

The most common cause is incorrect NordVPN service credentials. If you see `AUTH_FAILED`, regenerate your credentials at **my.nordaccount.com → NordVPN → Manual setup → Service credentials**, update `docker-compose.yml`, and run:

```bash
docker compose up -d gluetun
docker compose up -d --force-recreate tsbridge-qbit
```

> **Important:** When gluetun is recreated, tsbridge-qbit must also be force-recreated (not just restarted) because it attaches to gluetun's network namespace by container ID. A plain restart will fail with "no such container".

---

## Program Setup: qBittorrent

Check the logs for the temporary admin password:

```bash
docker logs qbittorrent
```

You will see something like:

```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: XXXXXXXX
```

Connect to:

```
https://qbittorrent.<tailnet>.ts.net
```

Go to **Tools → Options → WebUI**:
- Change username and password
- Under **Security**, disable **Host header validation** — this is required for tsbridge to proxy requests without getting a connection reset

Go to **Tools → Options → Downloads → Saving Management**:
- Default Torrent Management Mode: `Automatic`
- When Torrent Category changed: `Relocate torrent`
- When Default Save Path Changed: `Switch affected torrents to Manual Mode`
- When Category Save Path Changed: `Switch affected torrents to Manual Mode`
- Tick both `Use Subcategories` and `Use Category paths in Manual Mode`
- Default Save Path: `/data/downloads/torrents`

In the left panel, right-click **Categories → All → Add category**:
- `movies` → Save Path: `movies`
- `tv` → Save Path: `tv`

Create categories **before** configuring download clients in Radarr/Sonarr.

---

## Program Setup: Prowlarr

Connect to:

```
https://prowlarr.<tailnet>.ts.net
```

Go to **Settings → Authentication** and set up a username and password.

Go to **Settings → Download Clients → +** and add qBittorrent:
- Host: `qbittorrent`
- Port: `8081`
- Untick `Use SSL`
- Username/password: your qBittorrent credentials
- Click **Test** (should get a green tick) then **Save**

---

## Program Setup: Radarr

Connect to:

```
https://radarr.<tailnet>.ts.net
```

Go to **Settings → Media Management → Add Root Folder**: set `/data/media/movies`

Enable **Use Hardlinks instead of Copy** (requires all paths on the same filesystem).

Go to **Settings → Download Clients → +** and add qBittorrent:
- Host: `qbittorrent`, Port: `8081`
- Untick `Use SSL`
- Category: `movies`
- Click **Test** then **Save**

Go to **Settings → General**, copy the API key, then in Prowlarr go to **Settings → Apps → + → Radarr**:
- Paste the API key
- Prowlarr Server: `https://prowlarr.<tailnet>.ts.net`
- Radarr Server: `https://radarr.<tailnet>.ts.net`
- Click **Test** then **Save**

---

## Program Setup: Sonarr

Connect to:

```
https://sonarr.<tailnet>.ts.net
```

Go to **Settings → Media Management → Add Root Folder**: set `/data/media/tv`

Enable **Use Hardlinks instead of Copy**.

Go to **Settings → Download Clients → +** and add qBittorrent:
- Host: `qbittorrent`, Port: `8081`
- Untick `Use SSL`
- Category: `tv`
- Click **Test** then **Save**

Go to **Settings → General**, copy the API key, then in Prowlarr go to **Settings → Apps → + → Sonarr**:
- Paste the API key
- Prowlarr Server: `https://prowlarr.<tailnet>.ts.net`
- Sonarr Server: `https://sonarr.<tailnet>.ts.net`
- Click **Test** then **Save**

---

## Program Setup: Jellyseerr

Connect to:

```
https://jellyseerr.<tailnet>.ts.net
```

Sign in using your Jellyfin account and configure it to connect to `https://jellyfin.<tailnet>.ts.net`
