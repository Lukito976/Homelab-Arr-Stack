Private Arr Stack with Gluetun (NordVPN) and tsbridge

This guide describes how to deploy a fully private Arr stack using Docker, where:
	•	Torrent traffic exits through NordVPN via Gluetun (OpenVPN)
	•	A kill switch prevents traffic leaks if the VPN drops
	•	No services expose ports to the LAN or WAN
	•	All web access happens only over Tailscale using tsbridge
	•	Arr applications communicate internally via Docker networking
	•	Media storage supports hardlinking for efficient imports

The stack includes:
	•	qBittorrent
	•	Prowlarr
	•	Sonarr
	•	Radarr
	•	Jellyseerr
	•	Gluetun
	•	tsbridge (two instances)

⸻

Architecture Overview

High-level design:
	•	qBittorrent runs inside Gluetun’s network namespace
	•	Gluetun handles all VPN routing and firewalling
	•	qBittorrent’s WebUI is bound to 127.0.0.1 only
	•	One tsbridge instance runs on the Docker network for Arr apps
	•	A second tsbridge instance runs on the host network for qBittorrent
	•	No application ports are exposed directly

Why two tsbridge containers are required:
	•	Docker DNS (sonarr, radarr, etc.) only works from within the Docker network
	•	127.0.0.1 access to Gluetun-published ports only works from the host network

⸻

Directory Layout

All paths must be on the same filesystem to allow hardlinks.

sudo mkdir -p /media/myfiles
sudo chown -R 1000:1000 /media/myfiles

Recommended structure:

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

docker-compose.yml

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

tsbridge Configuration

tsbridge-arr.toml

Used for Arr applications on the Docker network.

[tailscale]
oauth_client_id = "REDACTED"
oauth_client_secret = "REDACTED"
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

[tailscale]
oauth_client_id = "REDACTED"
oauth_client_secret = "REDACTED"
state_dir = "/var/lib/tsbridge"
default_tags = ["tag:tsbridge"]

[[services]]
name = "qbittorrent"
backend_addr = "http://127.0.0.1:8080"


⸻

Arr Application Configuration

qBittorrent (in Sonarr and Radarr)
	•	Host: gluetun
	•	Port: 8080
	•	Categories:
	•	tv
	•	movies

Sonarr
	•	Root folder: /data/media/tv
	•	Download folder: /data/downloads/torrents/tv

Radarr
	•	Root folder: /data/media/movies
	•	Download folder: /data/downloads/torrents/movies

Prowlarr
	•	Add indexers
	•	Add Sonarr and Radarr as applications
	•	Sync indexers

⸻

Verification

Confirm VPN egress

docker exec -it gluetun curl -s https://ifconfig.me/ip

The IP should not match your ISP.

Confirm no LAN exposure

sudo ss -tulpn | grep 8080

Expected result:
	•	Bound to 127.0.0.1:8080

Confirm qBittorrent uses Gluetun networking

docker inspect qbittorrent --format '{{.HostConfig.NetworkMode}}'

Expected output:

service:gluetun

Confirm Arr connectivity

Use “Test” in Sonarr and Radarr download client settings. The test should succeed.

⸻

Why This Works
	•	Gluetun provides VPN routing and firewall enforcement
	•	qBittorrent cannot leak traffic outside the VPN
	•	tsbridge provides HTTPS access over Tailscale only
	•	No service ports are reachable from the local network
	•	Docker networking is preserved for Arr inter-service communication
	•	A single filesystem enables instant imports via hardlinks

⸻

Result
	•	Torrent traffic protected from ISP exposure
	•	Access limited to authenticated Tailnet devices
	•	No accidental LAN or WAN exposure
	•	Efficient storage usage with hardlinks
	•	Clear separation of concerns between networking layers

This setup represents a stable, reproducible pattern for a private Arr stack in a homelab environment.
