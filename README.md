# Homelab: Private Arr Stack (qBittorrent + Gluetun + tsbridge)

This repository documents how to deploy a private, VPN-protected Arr stack using Docker with:
*	**[qBittorrent](https://github.com/linuxserver/docker-qbittorrent)** for torrent downloading.
*	**[Gluetun](https://github.com/qdm12/gluetun)** to route all torrent traffic through NordVPN (OpenVPN) with a kill switch.
*	**[Prowlarr](https://github.com/linuxserver/docker-prowlarr)** for indexer management.
*	**[Radarr](https://github.com/linuxserver/docker-radarr)** for movie automation.
*	**[Sonarr](https://github.com/linuxserver/docker-sonarr)** for TV automation.
*	**[Jellyseerr](https://hub.docker.com/r/fallenbagel/jellyseerr/)** for media requests.
*	**[tsbridge](https://github.com/jtdowney/tsbridge)** to securely expose services over Tailscale.
*	No public ports and no LAN/WAN exposure

All external access is restricted to devices on your Tailnet.

---

## Prerequisites

* Linux server (Tested on [Ubuntu 24.04.3 LTS](https://ubuntu.com/download/server))
* [Docker](https://docs.docker.com/engine/install/ubuntu/) + Docker Compose installed
* A [Tailscale](https://tailscale.com/) account
* A Tailscale OAuth client with tag permissions
* A [NordVPN](https://refer-nordvpn.com/vHqBPZtMuUN) subscription with service credentials enabled
* This guide assumes that you have either configured Jellyfin, or have followed my [Homelab Program Setup](https://github.com/Lukito976/Homelab-Program-Setup-via-tsbridge)

---

## Directory Layout

## Configuration files

```
~/arr-stack
├── docker-compose.yml
├── tsbridge-arr.toml
└── tsbridge-qbit.toml
```

## Media and application data

**All paths must be on the same filesystem to allow hardlinking.**

```
/media/myfiles
├── appdata
│   ├── gluetun
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

All program files & downloads will live here:

```bash
mkdir -p ~/arr-stack
cd ~/arr-stack
```
---

## Step 2 — Configure Tailscale ACLs

Because tsbridge uses OAuth with tags, your Tailnet must allow the tag. *If you followed my previous guide, you can skip this step and reuse the OAuth credentials you have already created.*


In **Tailscale Admin → Access Controls**, ensure:

```json
{
  "tagOwners": {
    "tag:tsbridge": ["YOUR_TAILSCALE_LOGIN_EMAIL"]
  }
}
```

Without this, tsbridge will fail to authenticate.

Next, generate a new a new OAuth Key:

Go to **Settings → Trust Credentials → + Credential (make sure to give it read and write privileges)**

Save the details of the key to populate the required files.

---

## Step 3 — Create tsbridge Configuration Files

Create `tsbridge-arr.toml` (Used for Arr applications on the Docker network).
  
```bash
nano ~/arr-stack/tsbridge-arr.toml
```

```toml
[tailscale]
oauth_client_id = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir = "/var/lib/tsbridge"
default_tags = ["tag:tsbridge"]

[[services]]
name = "prowlarr"
backend_addr = "http://prowlarr:9696"

[[services]]
name = "sonarr"
backend_addr = "http://sonarr:8989"

[[services]]
name = "radarr"
backend_addr = "http://radarr:7878"

[[services]]
name = "jellyseerr"
backend_addr = "http://jellyseerr:5055"
```

Create `tsbridge-qbit.toml` (Used only for qBittorrent via localhost).
  
```bash
nano ~/arr-stack/tsbridge-qbit.toml
```

```toml
[tailscale]
oauth_client_id = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir = "/var/lib/tsbridge"
default_tags = ["tag:tsbridge"]

[[services]]
name = "qbittorrent"
backend_addr = "http://127.0.0.1:8080"
```

---

## Step 4 — Create docker-compose.yml

Create the file:

```bash
nano ~/arr-stack/docker-compose.yml
```

```yaml
services:
  tsbridge-arr:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge-arr
    command: ["-config", "/config/tsbridge.toml"]
    volumes:
      - tsbridge-arr-state:/var/lib/tsbridge
      - ./tsbridge-arr.toml:/config/tsbridge.toml:ro
    restart: unless-stopped

  tsbridge-qbit:
    image: ghcr.io/jtdowney/tsbridge:latest
    container_name: tsbridge-qbit
    network_mode: host
    command: ["-config", "/config/tsbridge.toml"]
    volumes:
      - tsbridge-qbit-state:/var/lib/tsbridge
      - ./tsbridge-qbit.toml:/config/tsbridge.toml:ro
    restart: unless-stopped

  gluetun:
    image: ghcr.io/qdm12/gluetun:latest
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=nordvpn
      - OPENVPN_USER=YOUR_NORD_SERVICE_USERNAME
      - OPENVPN_PASSWORD=YOUR_NORD_SERVICE_PASSWORD
      - SERVER_COUNTRIES=United States
      - TZ=Etc/UTC
    ports:
      - "127.0.0.1:8081:8081" #NOTE I map qBittorrent to port 8081 since I have Nextcloud AIO mapped to port 8080 already. The default configuration for qBittorrent is port 8080
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8081 #See above
    volumes:
      - /media/myfiles/appdata/qbittorrent:/config
      - /media/myfiles:/data
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/myfiles/appdata/prowlarr:/config
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/myfiles/appdata/radarr:/config
      - /media/myfiles:/data
    restart: unless-stopped

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
    volumes:
      - /media/myfiles/appdata/sonarr:/config
      - /media/myfiles:/data
    restart: unless-stopped

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - TZ=Etc/UTC
    volumes:
      - /media/myfiles/appdata/jellyseerr:/app/config
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

## Step 5 — Start the Stack

Run the following:

```bash
cd ~/arr-stack
docker compose up -d
```

Verify qBittorrent is not LAN-exposed:

```bash
sudo ss -tulpn | grep 8081
```

Expected result:

```
127.0.0.1:8081
```

For further verification, run these commands:

```bash
curl -s https://ifconfig.me/ip && echo
docker exec -it gluetun wget -qO- https://ifconfig.me/ip && echo
```

If you get two different outputs like this:
  
```
97.x.x.x
181.x.x.x
```

That means that the VPN is working that the torrent will read the latter. If you see the former repeated twice, the VPN is not working.

---

Program Setup: qBittorrent

Access via Tailnet:

https://qbittorrent.<tailnet>.ts.net

Confirm:
	•	Web UI loads
	•	Torrents download successfully
	•	External IP matches VPN, not ISP

⸻

Program Setup: Prowlarr

https://prowlarr.<tailnet>.ts.net

	•	Add indexers
	•	Configure applications:
	•	Sonarr
	•	Radarr
	•	Sync indexers

⸻

Program Setup: Sonarr

https://sonarr.<tailnet>.ts.net

	•	Root folder:

/data/media/tv


	•	Download folder:

/data/downloads/torrents/tv



Configure qBittorrent download client:
	•	Host: gluetun
	•	Port: 8080
	•	Category: tv

⸻

Program Setup: Radarr

https://radarr.<tailnet>.ts.net

	•	Root folder:

/data/media/movies


	•	Download folder:

/data/downloads/torrents/movies



Configure qBittorrent download client:
	•	Host: gluetun
	•	Port: 8080
	•	Category: movies

⸻

Program Setup: Jellyseerr

https://jellyseerr.<tailnet>.ts.net

	•	Connect to Sonarr and Radarr
	•	Configure request permissions

⸻




service:gluetun


⸻

Result

This setup provides:
	•	VPN-protected torrent traffic with kill switch
	•	No LAN or WAN service exposure
	•	Tailnet-only HTTPS access
	•	Functional Arr automation
	•	Hardlink-based imports for efficiency
	•	Clear separation of networking responsibilities

This represents a stable, reproducible pattern for running a private Arr stack in a homelab environment.
