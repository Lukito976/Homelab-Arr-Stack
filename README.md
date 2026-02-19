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

## Program Setup: qBittorrent

Check logs for qbittorrent container
```bash
sudo docker logs qbittorrent
```
You will see in the logs something like:
```
The WebUI administrator username is: admin
The WebUI administrator password was not set. A temporary password is provided for this session: <your-password-will-be-here>
```
Now you can go to URL: 
```
https://qbittorrent.<tailnet>.ts.net
```

Go to `Tools - Options - WebUI` - you can change the user and password here but remember to scroll down and save it.
In left panel go to Categories - All - right click and 'add category':
For Radarr: `Category: movies`
`Save Path: movies` (this will be appended to '/data/torrents/ Default Save Path you set above)
For Sonarr: `Category: tv`
`Save Path: tv`

Create categories first and only then configure the steps below, as doing it opposite way round caused the Categories to disappear

With categories created - go to -  `Tools - Options - Downloads` and in `Saving Management` make sure your settings match [THIS](https://trash-guides.info/Downloaders/qBittorrent/How-to-add-categories/)
So `Default Torrent Management Mode - Automatic`
`When Torrent Category changed - Relocate torrent` 
`When Default Save Path Changed - Switch affected torrents to Manual Mode` 
`When Category Save Path Changed - Switch affected torrents to Manual Mode` 
Tick BOTH BOXES for `Use Subcategories` and `Use Category paths in Manual Mode` (NOT shown on Trash Guides) 
Default Save Path: - set to `/data/torrents` (so it matches your folder structure) - then scroll down and `Save`.
On Trash Guides it shows `Copy .torrent files to` but its optional, you can leave it blank

## Program Setup: Prowlarr

Connect to URL: 
```
https://prowlarr.<tailnet>.ts.net
```

Now configure Prowlarr service (each of these services will require to set up user/pass):
Use 'Form (login page) authentication and set your user and pass for all.

Go to `Settings - Download Clients` - `+` symbol - Add download client - choose `qBittorrent` (unless you decided touse different download client)
UNTICK the `Use SSL` (unless you have SSL configured in qBittorrent - Tools - Options -WebUI but by default it is not used)
Host - use `qbittorrent` and port - put the port id matching the WebUI in docker-compose for qBittorrent (default is `8080`)
username and password - use the one that you configured for qBittorrent in previous step
Click little `Test` button at the bottom, make sure you get a green `tick` then `Save`.

## Program Setup: Radarr

Connect to URL: 
```
https://radarr.<tailnet>.ts.net
```

Go to `Settings - Media Management - Add Root Folder` (scroll down to the bottom) - set  `/data/media/movies` as your root folder
Still in `Settings - Media Management - click Show Advanced - Importing - Use Hardlinks instead of Copy` - make sure its 'ticked'

Optional - you can also tick `Rename Movies` and `Delete empty movie folders during disk scan` , and in `Import Extra Files` - make sure that box is ticked
and in `Import Extra files` field type `srt,sub,nfo` (those 3 changes are all optional)

Then `Settings- Download clients` - click `plus` symbol, choose `qBittorrent` etc - basically same steps as for Prowlarr
so Host `qbittorrent`, port `8080`, ,make sure SSL is unticked, username admin and password - one you configured for qBittorrent
and change the Category to `movies` (needs to match qbittorrent Category)
Now click the `Test` and if you have green 'tick' - `Save`.
Now go to `Settings - General` - scroll down to API key - Copy API key - go back to `Prowlarr - Settings - Apps` -click `+` - Radarr - paste  API key.
Then change `Prowlarr Server` to `https://prowlarr.<tailnet>.ts.net` and `Radarr Server` to `https://radarr.<tailnet>.ts.net`
Click `Test` and if Green - Save

## Program Setup: Sonarr

Connect to URL: 
```
https://sonarr.<tailnet>.ts.net
```

Go to `Settings - Media Management - Add Root Folder` (scroll down to the bottom) - set  `/data/media/tv` as your root folder
Still in `Settings - Media Management - click Show Advanced - Importing - Use Hardlinks instead of Copy` - make sure its 'ticked'

Optional - you can also tick `Rename Episodes` and `Delete empty Folders - delete empty series and season folders during disk scan` <br />
Then in `Import Extra Files` - make sure that box is ticked and in `Import Extra files` field type `srt,sub,nfo` (those 3 changes are all optional)

Then `Settings- Download clients` - click `plus` symbol, choose `qBittorrent` etc - basically same steps as for Prowlarr
so Host `qbittorrent`, port `8081`, ,make sure SSL is unticked, username admin and password - one you configured for qBittorrent
and change the Category to `tv` (needs to match qbittorrent Category)
Now click the `Test` and if you have green 'tick' - `Save`.
Now go to `Settings - General` - scroll down to API key - Copy API key - go back to `Prowlarr - Settings - Apps` -click `+` - Sonarr - paste  API key.
Then change `Prowlarr Server` to `https://prowlarr.<tailnet>.ts.net` and `Sonarr Server` to `https://sonarr.<tailnet>.ts.net`
Click `Test` and if Green - Save

## Program Setup: Jellyseer

Connect to URL: 
```
https://jellyseer.<tailnet>.ts.net
```

Sign in using your Jellyfin account, and configure it to connect to your jellyfin tailnet `https://jellyfin.<tailnet>.ts.net`
