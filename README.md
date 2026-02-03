Below is the clean, GitHub-ready README, rewritten to match the exact formatting, tone, and structure of the example you provided.
It reflects the current, working Arr + Gluetun + tsbridge architecture and is safe to paste directly into a repository.

⸻

Homelab: Private Arr Stack (qBittorrent + Gluetun + tsbridge)

This repository documents how to deploy a private, VPN-protected Arr stack using Docker with:
	•	qBittorrent for torrent downloading
	•	Gluetun to route all torrent traffic through NordVPN (OpenVPN) with a kill switch
	•	Prowlarr for indexer management
	•	Sonarr for TV automation
	•	Radarr for movie automation
	•	Jellyseerr for media requests
	•	tsbridge to securely expose services over Tailscale
	•	No public ports and no LAN/WAN exposure

All external access is restricted to devices on your Tailnet.

⸻

Prerequisites
	•	Linux server (tested on Ubuntu 24.04 LTS)
	•	Docker + Docker Compose installed
	•	A Tailscale account
	•	A Tailscale OAuth client with tag permissions
	•	A NordVPN subscription with service credentials enabled

⸻

Directory Layout

Configuration files

~/arr-stack
├── docker-compose.yml
├── tsbridge-arr.toml
└── tsbridge-qbit.toml

Media and application data

All paths must be on the same filesystem to allow hardlinking.

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


⸻

Step 1 — Install Docker

If Docker is not installed, follow the official instructions:

curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

Verify:

docker version
docker compose version
docker run hello-world


⸻

Step 2 — Create the Media Directory

sudo mkdir -p /media/myfiles
sudo chown -R 1000:1000 /media/myfiles


⸻

Step 3 — Create the Arr Stack Directory

mkdir -p ~/arr-stack
cd ~/arr-stack


⸻

Step 4 — Configure Tailscale ACLs

Because tsbridge uses OAuth with tags, your Tailnet must allow the tag.

In Tailscale Admin → Access Controls, ensure:

{
  "tagOwners": {
    "tag:tsbridge": ["YOUR_TAILSCALE_LOGIN_EMAIL"]
  }
}

Next, create a new OAuth client with read and write permissions and save:
	•	OAuth Client ID
	•	OAuth Client Secret

⸻

Step 5 — Create tsbridge Configuration Files

tsbridge-arr.toml

Used for Arr applications on the Docker network.

nano ~/arr-stack/tsbridge-arr.toml

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


⸻

tsbridge-qbit.toml

Used only for qBittorrent via localhost.

nano ~/arr-stack/tsbridge-qbit.toml

[tailscale]
oauth_client_id = "YOUR_OAUTH_CLIENT_ID"
oauth_client_secret = "YOUR_OAUTH_CLIENT_SECRET"
state_dir = "/var/lib/tsbridge"
default_tags = ["tag:tsbridge"]

[[services]]
name = "qbittorrent"
backend_addr = "http://127.0.0.1:8080"


⸻

Step 6 — Create docker-compose.yml

nano ~/arr-stack/docker-compose.yml

services:
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
      - "127.0.0.1:8080:8080"
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
      - WEBUI_PORT=8080
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

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    environment:
      - TZ=Etc/UTC
    volumes:
      - /media/myfiles/appdata/jellyseerr:/app/config
    restart: unless-stopped

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

volumes:
  tsbridge-arr-state:
  tsbridge-qbit-state:


⸻

Step 7 — Start the Stack

cd ~/arr-stack
docker compose up -d

Verify qBittorrent is not LAN-exposed:

sudo ss -tulpn | grep 8080

Expected result:

127.0.0.1:8080


⸻

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

Verification

Confirm VPN egress:

docker exec -it gluetun curl -s https://ifconfig.me/ip

Confirm qBittorrent networking:

docker inspect qbittorrent --format '{{.HostConfig.NetworkMode}}'

Expected:

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
