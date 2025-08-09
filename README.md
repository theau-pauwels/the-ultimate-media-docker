# The Ultimate Media Docker

**The Ultimate Media Docker** is a modular Docker Compose stack for a self-hosted media NAS on Ubuntu (x86_64). It bundles multiple services for automated media downloading, management, and streaming, all behind a secure VPN and reverse proxy. This stack uses Traefik for smart routing (supporting both subdomain and path-based access), includes a kill-switch via WireGuard (Gluetun), and organizes all persistent data under a single directory (`/srv/media-stack`) for convenience.
## Included Services

- **Gluetun (VPN Client):** Routes traffic for specific containers through a WireGuard VPN tunnel (protects download traffic and provides a kill-switch)
- **qBittorrent (Downloader):** BitTorrent client for downloading media (web UI secured behind VPN)
- **Sonarr (TV Automation):** Monitors TV show RSS/indexers and sends download requests for new episodes
- **Radarr (Movies Automation):** Monitors movie indexers and sends download requests for new movie releases
- **Lidarr (Music Automation):** Manages music library by fetching new album releases from indexers
- **Prowlarr (Indexer Manager):** Central manager for indexers (search providers) for Sonarr/Radarr/Lidarr
- **Bazarr (Subtitles Manager):** Finds and downloads subtitles for movies and TV shows in your library
- **Jellyfin (Media Server):** Streams your video/music collection to web browsers and apps (open-source alternative to Plex)
- **Jellyseerr (Requests UI):** Web application for users to request new movies or shows (integrates with Jellyfin/Sonarr/Radarr)
- **Prometheus (Monitoring):** Metrics collector for system, container, and application statistics
- **Grafana (Dashboards):** Visualization UI for metrics (comes preconfigured for Prometheus data source)
- **Node Exporter (Host Metrics):** Exposes Linux host hardware/OS metrics to Prometheus (CPU, memory, disk, etc.)
- **cAdvisor (Container Metrics):** Provides Docker container resource usage metrics to Prometheus
- **Traefik (Reverse Proxy):** Routes HTTP(S) traffic to services, handles SSL via Let’s Encrypt, and enables both subdomain and URL path routing
- **Autoheal (Watchdog):** Monitors all containers and automatically restarts any container that reports an unhealthy status
- **Watchtower (Auto-Updates):** Monitors Docker images and automatically updates containers to the latest image versions

All web services are routed through Traefik, which means you can access them via friendly URLs (with optional HTTPS). For example, Jellyfin might be available at `https://jellyfin.your-domain.com` (subdomain) or `https://your-domain.com/jellyfin` (path-based). The qBittorrent UI is also behind Traefik and the VPN, preventing direct exposure. The stack is designed such that if the VPN goes down, qBittorrent’s network access is cut off (kill-switch), protecting your privacy.

## File Structure
After cloning this repository and preparing the environment (see below), your setup will consist of the following structure on the host:

```
the-ultimate-media-docker/   # <- Git repository files (compose YAMLs, README, etc.)
├── docker-compose.yml               # Primary Compose file (aggregates others)
├── compose.proxy.yml                # Traefik reverse proxy service
├── compose.arr.yml                  # *Arr services (Sonarr, Radarr, etc.)
├── compose.media.yml                # Media services (Jellyfin, Jellyseerr)
├── compose.dl.yml                   # Download services (Gluetun VPN + qBittorrent)
├── compose.monitoring.yml           # Monitoring services (Prometheus, Grafana, etc.)
├── compose.utils.yml                # Utility services (Autoheal, Watchtower)
├── .env                             # Environment variables (configuration options)
├── secrets/
│   └── admin_credentials.env        # Default admin credentials (to be changed)
└── vpn/
  └── wg0.conf                     # WireGuard configuration (to be provided by user)
```

All persistent data is stored in `/srv/media-stack` on the host. Within that directory, subfolders are used to organize configs and media:

```
/srv/media-stack/
├── config/
│   ├── sonarr/               # Sonarr config & database
│   ├── radarr/               # Radarr config & database
│   ├── lidarr/               # Lidarr config & database
│   ├── prowlarr/             # Prowlarr config
│   ├── bazarr/               # Bazarr config
│   ├── jellyfin/             # Jellyfin config & library data
│   ├── jellyseerr/           # Jellyseerr config & database
│   ├── qbittorrent/          # qBittorrent config (qBittorrent.conf, etc.)
│   ├── prometheus/           # Prometheus data (time-series database)
│   ├── grafana/              # Grafana data (database, etc.)
│   └── traefik/              # Traefik config (acme.json for certificates)
├── media/
│   ├── movies/               # Library of movie files (Radarr will put movies here)
│   ├── tv/                   # Library of TV show files (Sonarr will put episodes here)
│   └── music/                # Library of music files (Lidarr will organize music here)
├── downloads/
│   ├── movies/               # Temporary download storage for movies
│   ├── tv/                   # Temporary download storage for TV
│   └── music/                # Temporary download storage for music
└── vpn/
    └── wg0.conf              # WireGuard config file for VPN (to be provided by user)
```

All services are configured to use these bind mounts, so you can easily back up your configuration and media by copying the `/srv/media-stack` directory.

Note: If you prefer a different base directory, you can change the `BASE_DIR` variable in the `.env` file (default is `/srv/media-stack`). All compose file volume paths will update accordingly.

## Prerequisites
- **Operating System:** Ubuntu 20.04/22.04 (or any Linux x86_64 with Docker support)
- **Docker Engine & Compose:** Ensure Docker and Docker Compose (v2) are installed on the host system. On Ubuntu, you can install Docker Engine and then enable the Compose Plugin.
- **Hardware:** At least 4 CPU cores and 6+ GB of RAM are recommended (the stack includes memory-intensive services like Jellyfin and Grafana). Ensure you have sufficient disk space for your media library and Docker images.

## Setup Instructions
### 1. Clone the Repository
Clone this GitHub project to your server, then enter the project directory:
```
git clone https://github.com/your-username/the-ultimate-media-docker.git
cd the-ultimate-media-docker
```
_(If you prefer not to use Git, you can instead download the repository as a ZIP and extract it.)_

### 2. Create the Media Directories
Create the bind mount directories on the host for configuration, media, and downloads. The default base path is `/srv/media-stack` (you can change this in the `.env` file if desired):
```
sudo mkdir -p /srv/media-stack/{config,media,downloads,monitoring,vpn}
sudo mkdir -p /srv/media-stack/config/{sonarr,radarr,lidarr,prowlarr,bazarr,jellyfin,jellyseerr,qbittorrent,prometheus,grafana,traefik}
sudo mkdir -p /srv/media-stack/media/{movies,tv,music}
sudo mkdir -p /srv/media-stack/downloads/{movies,tv,music}
sudo mkdir -p /srv/media-stack/monitoring/{prometheus,grafana}
sudo touch /srv/media-stack/config/traefik/acme.json
```
These commands will set up all necessary subfolders. We also create an empty `acme.json` file for Traefik (to store HTTPS certificates). 

**Set Permissions:** Ensure the directories are owned by the user/group that will run the Docker containers. By default, the stack expects UID 1000 and GID 1000 (the first user on many systems). If that’s your user, adjust ownership and permissions:
```
sudo chown -R 1000:1000 /srv/media-stack
sudo chmod -R 775 /srv/media-stack
sudo chmod 600 /srv/media-stack/config/traefik/acme.json
```
The `acme.json` file must have permissions `600` so Traefik can write certificates to it. The rest of the directories are given group-writable (`775`) so containers (if run under a specific user/group) can write as needed. 

If your user or service account has a different UID/GID than 1000, update the `PUID` and `PGID` values in `.env` accordingly, and ensure the directories are owned by that UID/GID.

### 3. Configure Environment Variables
Open the `.env` file in a text editor. This file contains configuration options you may want to adjust. It’s pre-filled with sensible defaults and examples:

```
# .env (excerpt)
PUID=1000                   # Unix user ID for containers to run as (match this to directory owner)
PGID=1000                   # Unix group ID for containers
TZ=UTC                      # Timezone, e.g., "America/New_York"
DOMAIN=example.com          # Your domain name for Traefik routing (e.g., "home.mydomain.net")
LETSENCRYPT_EMAIL=me@example.com   # Email for Let's Encrypt SSL certificate registration
#BASE_DIR=/srv/media-stack   # Base directory for all data (uncomment to override default path)
```

Key settings to review in `.env`:
- **PUID / PGID:** Ensure these match the user and group that should own the media files. If you created `/srv/media-stack` as a different user, update accordingly. This ensures the Docker containers don’t run as root but as your user, to avoid permission issues.
- **TZ:** Set this to your timezone (for accurate timestamps and schedule alignment).
- **DOMAIN:** Set this to the domain name you plan to use for accessing your services. This can be a root domain (e.g., `example.com`) or a subdomain (e.g., `home.example.com`). Traefik will use this for routing rules. **Important:** You should have DNS records for the services you intend to expose (or a wildcard) pointing to your server’s IP if you plan to access them externally.
- **LETSENCRYPT_EMAIL:** _(Optional but recommended if using HTTPS)_ Your email for Let’s Encrypt to notify about certificate expiration issues. If you don’t plan to use HTTPS/Let’s Encrypt, you can leave this blank, but then adjust Traefik settings to avoid requesting certificates (see **HTTPS and Traefik below**).
The `.env` file also contains the `COMPOSE_FILE` definition that combines all the compose YAML files. By default, **all services are enabled**. If you wish to exclude any component, you can edit the `COMPOSE_FILE` list or use Docker Compose profiles (not configured by default in this stack). For most users, the default will be fine.

### 4. Provide VPN Credentials
You need to supply your own WireGuard configuration for the VPN client (Gluetun). This stack expects a WireGuard config file at `vpn/wg0.conf`.

**Replace the placeholder file:** Open `vpn/wg0.conf` in an editor – by default it contains only a placeholder and comments. Delete its contents and paste your WireGuard client configuration. This should be the `.conf` file provided by your VPN service or your own WireGuard server, typically containing the `[Interface]` and `[Peer]` sections with keys, addresses, and endpoint. For example, a WireGuard config might look like:
```
[Interface]
PrivateKey = <YOUR_PRIVATE_KEY>
Address = 10.13.37.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = <VPN_SERVER_PUBLIC_KEY>
Endpoint = <VPN_SERVER_HOST>:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

After adding your actual config, save the file. Make sure the file has correct permissions so it’s not world-readable (it contains your VPN keys):
`sudo chmod 600 /srv/media-stack/vpn/wg0.conf`
_(The compose file will mount this file into the Gluetun container to establish the VPN tunnel.)_ 

**VPN Provider Compatibility:** Gluetun supports many VPN providers. If your provider gave you an OpenVPN `.ovpn` instead of WireGuard, you have two options: either convert to WireGuard (if the provider supports it) or adjust the Gluetun settings to use OpenVPN (beyond the scope of this guide). This stack is configured for WireGuard by default (for better performance).

### 5. Review Default Admin Credentials
The file `secrets/admin_credentials.env` contains default login credentials for certain web services:
```
# Default credentials (CHANGE THEM ASAP!)
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=OTaEe2NeFpMw26Ju
WEBUI_USER=admin
WEBUI_PASS=ZPBgS1SyiW1Jvdv6
```

- **Grafana:** Uses the `GF_SECURITY_ADMIN_USER` and `GF_SECURITY_ADMIN_PASSWORD`. By default, it’s set to `admin` / (the random string above). **You should change the password** after first login (Grafana’s UI allows changing the admin password). Alternatively, you can edit the `admin_credentials.env` file now to set a custom password before deploying.
- **qBittorrent Web UI:** Uses WEBUI_USER and WEBUI_PASS for the web interface login. It’s set to admin / (random string). You can modify it in the admin_credentials.env file or change it via qBittorrent’s settings after deployment.
- **Other services:** Sonarr, Radarr, Lidarr, Prowlarr, and Bazarr do **not** have built-in authentication by default (they rely on network security or a reverse proxy auth). Jellyfin and Jellyseerr will prompt you to create admin users on first launch (they are not pre-set here). We **highly recommend** securing these UIs either by enabling authentication in the apps or using Traefik middlewares (e.g., basic auth) if you expose them externally.

**IMPORTANT:** The generated passwords above are random defaults included for convenience. Since this is a public repository, assume these credentials are known to others. **Change all default passwords** to something unique and strong **before** exposing the services on the internet. You can edit the `secrets/admin_credentials.env` file or update credentials via each application’s interface after launching.

### 6. Launch the Stack
With all configurations in place, you’re ready to start the stack. From the project directory, run:
```
docker compose up -d
```

This will bring up all the services in detached mode. Docker will pull the necessary images (this may take some time on the first run, as images like Jellyfin or Grafana can be large). 

You should see Docker Compose creating the network `media-stack` and then starting each container. Verify that no containers immediately exit or crash:
```
docker compose ps
```

All containers should be in the “Up (healthy)” state after a couple of minutes. Note that some services (e.g., Prometheus, Grafana) might not have healthchecks and will show just “Up”. The Autoheal container will monitor any with a healthcheck and restart them if needed.

### 7. Post-Launch Configuration
Once everything is up and running, there are a few important post-install steps to finalize your setup:

#### A. Traefik & DNS: 
If you set `DOMAIN` in `.env`, Traefik will attempt to route using that domain. Ensure your DNS records are configured. For example, if `DOMAIN=example.com`, and you want to use subdomains for services:
Create DNS A (or CNAME) records for each service: `sonarr.example.com`, `radarr.example.com`, etc., pointing to your server’s IP. Alternatively, a wildcard DNS record for `*.example.com` can cover all subdomains.
If you plan to use path-based routes (e.g., `example.com/sonarr`), ensure DNS for the root/base domain is set up.

Traefik is configured to automatically obtain Let’s Encrypt SSL certificates for any service domain it routes (using the TLS-ALPN challenge on port 443). The first time you access a service via HTTPS, Traefik will negotiate obtaining a certificate. The certificates are stored in `config/traefik/acme.json`. If Traefik fails to get a certificate (e.g., DNS not pointing correctly, or port 443 blocked), it will serve the services on HTTP (port 80) without encryption by default. 

If you **do not** have a domain or don’t want HTTPS, you should disable Let’s Encrypt in Traefik to avoid errors:

Open `compose.proxy.yml` and remove or comment out the lines under traefik service that configure the certificate resolver. Specifically, remove the `--certificatesresolvers.le...` lines and the redirect of entrypoint 80 to 443.
Also consider setting `DOMAIN` to something like `localhost` in `.env` and access via `http://<your-server-ip>:8080` by mapping ports, or simply use the server’s LAN IP and define host rules accordingly. (This is an advanced scenario – for most users we assume a domain and desire for HTTPS.)

#### B. Application Base URLs (for path routing): 
This stack supports both subdomain and path-based routing. If you plan to access services at subdomains (e.g., `radarr.example.com`), you generally don’t need to change any in-app settings. However, if you want to access them at a sub-path of a domain (e.g., `example.com/radarr`), you **must configure each application to know its “base URL”**:

**Sonarr/Radarr/Lidarr/Prowlarr:** Go to **Settings > General** (enable advanced settings) and set the URL Base to `/sonarr`, `/radarr`, `/lidarr`, `/prowlarr` respectively. Save changes. This ensures the web app generates correct URLs for assets and APIs when behind a subfolder. After setting this, you will need to re-access the app at the new URL (e.g., `http(s)://example.com/sonarr`).

**Bazarr:** Similarly, in Bazarr settings, set the base URL to `/bazarr`.

**Jellyfin:** Navigate to **Dashboard > Networking** and set a Base URL (e.g., `jellyfin`) if you intend to use `example.com/jellyfin`. Jellyfin will then expect to be accessed under that path.

**Jellyseerr:** During initial setup of Jellyseerr, you can specify the application URL. If using a sub-path, include it there (e.g., `https://example.com/jellyseerr`). In the settings, you may also find an option for base URL or external URL to update.

**Grafana:** Grafana is configured via environment variable to know its root URL if behind a subpath. In our setup, we default to subdomain (e.g., `grafana.example.com`). If you want `example.com/grafana`, you should modify `GF_SERVER_ROOT_URL` in the compose file or via an env var. (By default, our config uses subdomain routing for Grafana.)

If you **only** use subdomains (which is simpler for most cases), you can ignore the base URL settings above. All services will work on subdomains out-of-the-box. 

#### C. Service Setup and Integration:
After deployment, you should visit each service’s web interface to complete setup:

**Sonarr/Radarr/Lidarr:** Add your media folder paths in each (they should see the host paths you mounted: for example, Sonarr will see a `/tv` folder for TV library and a `/downloads` folder for incoming files). Configure indexers by integrating with Prowlarr (Prowlarr can generate a Sonarr/Radarr API key and URL for you to add in Sonarr/Radarr under Indexers > Add > Torznab). Also add the qBittorrent download client in Sonarr/Radarr (Settings > Download Clients): use host `gluetun` and port `8080` with the credentials from `WEBUI_USER`/`WEBUI_PASS` (see VPN & Network below for details).

**Prowlarr:** Add your torrent indexers (and/or usenet indexers if you use them). Under “Apps”, connect Prowlarr to Sonarr/Radarr/Lidarr so that found releases can be sent to them.

**Bazarr:** Connect Bazarr to Sonarr and Radarr (via their APIs) so it knows what shows/movies you have and can fetch subtitles. Also configure your subtitle providers (add OpenSubtitles credentials, etc.).

**Jellyfin:** Complete the initial wizard by creating an admin user and adding media libraries. Point the libraries to `/media/movies`, `/media/tv`, `/media/music` as appropriate (these correspond to the host’s `/srv/media-stack/media/...` directories).

**Jellyseerr:** Follow the setup to connect Jellyseerr to your Jellyfin server (you’ll need the Jellyfin URL and an API key from Jellyfin’s dashboard) and to Sonarr/Radarr (for fulfilling requests).

**Grafana:** Access Grafana (default user: `admin`, pass: whatever is set in `admin_credentials.env`). A Prometheus data source should be added automatically (if not, add a new data source for Prometheus pointing to `http://prometheus:9090`). You can then import or create dashboards for system monitoring. For example, you might import official dashboards for Node Exporter and cAdvisor to get CPU/Memory/Network graphs for your system and containers.

#### D. VPN & Network Behavior: 
The qBittorrent container is strictly routed through the VPN container (Gluetun). This means:

qBittorrent’s web UI is not directly exposed on the host network. It’s only reachable through Traefik via the gluetun container. We configured Traefik to route `qbittorrent.<DOMAIN>` (and `<DOMAIN>/qbittorrent`) to qBittorrent’s interface **through** the VPN container.
If the VPN connection is down (or Gluetun is unhealthy), qBittorrent will have no internet connectivity. This is by design (kill-switch). The web UI might still load, but no downloads will happen until the VPN is back up.
Sonarr/Radarr connect to qBittorrent internally. In Sonarr’s download client settings, use `gluetun` as the host and `8080` as port (since Sonarr and Gluetun are on the same Docker network, Sonarr can reach qBittorrent’s interface via the Gluetun container’s hostname). We set a network alias “qbittorrent” on the Gluetun service for convenience, so you can alternatively use qbittorrent as the host name. The username/password are what you set in WEBUI_USER/WEBUI_PASS.
You do not need to open qBittorrent’s port on your router or host – incoming torrent connections are handled via the VPN’s port forwarding (if your VPN provider supports it). Gluetun can manage VPN port forwarding (for providers like PIA) automatically. If using a custom WireGuard server or a provider that doesn’t support port forwarding, you might not get incoming connections – which can slow torrenting. In that case, consider enabling DHT and using well-seeded torrents or switching to a provider that supports port forwarding.

#### E. Media Import and Hardlinks: 
This setup separates **download** locations and final **media** locations. Sonarr and Radarr will move files from the `downloads` directories to the `media` library after download (and optionally delete or hardlink). By default, the volumes for `/downloads` and `/movies`/`/tv` in the container are separate mounts, which means **hardlinks will not work** (the files will be copied instead of hardlinked). If you prefer to use hardlinks to save space (i.e., not duplicate data when importing), you need to ensure the downloads and media directories are on the same filesystem mount. Since we have them under the same base `/srv/media-stack`, if that is a single filesystem, it actually can support hardlinks – however, Docker’s separate volume mount may still treat them as distinct. To use hardlinks:
One approach is to modify the compose files to mount a common parent directory for both downloads and media in Sonarr/Radarr (so they see both paths under one volume). For simplicity, our default config does not do this. Instead, we recommend disabling hardlinks in Sonarr/Radarr: In Sonarr/Radarr settings, go to Media Management and uncheck “Use Hardlinks instead of Copy”. This will ensure Sonarr/Radarr will copy and then delete original files.
If you wish to enable hardlinking, you could mount /srv/media-stack as /data in the containers and have subfolders (e.g. `/data/media/movies` and `/data/downloads/movies`). This requires adjusting the volume mounts. (Advanced users only – the current setup is perfectly functional with copies.)

#### F. Monitoring Dashboards: 
Once Prometheus and Grafana are running, you can import premade dashboards to visualize your system:

**Node Exporter (Host metrics):** Import Prometheus’ official Node Exporter dashboard (available on Grafana’s Dashboards site) to see CPU, memory, disk, etc.

**cAdvisor (Docker container metrics):** There are community dashboards that show per-container CPU/RAM usage, I/O, network, etc. You should be able to find one on Grafana’s dashboard repository (for example, search for “Docker Metrics by cAdvisor”).

**Traefik metrics:** By default, we haven’t enabled Traefik metrics, but you can turn them on in Traefik’s config if desired and then import a Traefik dashboard.

#### G. Jellyfin Setup: 
In Jellyfin’s dashboard, you may want to set up hardware acceleration for transcoding (if your server has a GPU and you use it). This usually involves passing through the GPU (e.g., via Docker `device` parameters which are not included by default in our compose). If you plan to do a lot of transcoding, research how to enable NVIDIA/Intel QuickSync in Jellyfin’s container. (This will require installing drivers on the host and modifying the compose file to add the device). For many users who stream in original format on a LAN, this might not be necessary.

### 8. Security Considerations
This stack is designed with security in mind, but **you** are responsible for the secure operation of your server. Keep these points in mind:

- **VPN (Privacy for Downloads):** qBittorrent’s traffic is entirely encapsulated in the VPN tunnel. This protects your peer-to-peer traffic from prying eyes. Ensure your VPN credentials/config remain secure. If the VPN disconnects, qBittorrent cannot reach the internet (so no accidental leaking of your IP). Gluetun’s healthcheck will mark it unhealthy if the VPN is down; Autoheal will attempt to restart it if that happens. If you see frequent restarts, investigate your VPN stability.

- **Firewall:** Even though qBittorrent’s web UI isn’t exposed directly, other services (Sonarr, Radarr, etc.) might be accessible through Traefik. If you’re accessing the services over the internet, consider restricting ports 80/443 on your router or using a VPN (the other kind – e.g., WireGuard or Tailscale for yourself) to access them, or implement authentication:

- You can add HTTP Basic Auth in Traefik for sensitive apps (Traefik dashboard, Sonarr, etc.) by adding middleware in Traefik’s config. For example, protect Traefik’s own dashboard or even the _*Arr apps with a password if you want an extra layer_ (since by default *Arr have no auth). Basic Auth middleware can be configured via labels on the service in the compose file or via a dynamic config file.

- Another approach is to keep these services accessible only on your LAN (don’t expose ports 80/443 to the internet) and use a VPN or SSH tunnel to access your home network remotely.

- **Auto-Updates (Watchtower):** Watchtower will automatically pull the latest Docker images for all containers and restart them with the existing configuration. This keeps you up-to-date with security patches and new features. However, automatic updates can occasionally break things if a new version changes something. It’s a good practice to monitor your containers (via logs or Grafana) so you notice if something goes wrong after an update. If you prefer to control updates manually, you can disable Watchtower by removing the service from `compose.utils.yml` or by stopping its container.

- **Autoheal:** Autoheal will restart containers that become unhealthy (as determined by their Docker healthchecks). For example, if the VPN container is unhealthy or if Jellyfin’s healthcheck fails (not configured by default to have one), they’ll restart. This helps keep your stack running smoothly without manual intervention. However, it’s not a substitute for investigating root causes. Frequent unhealthy restarts could indicate a problem (e.g., an unreliable VPN server or a misconfiguration).

- **Admin Interfaces:** As mentioned, change all default passwords. Don’t leave the Grafana admin with the default password, and similarly for qBittorrent. For Jellyfin/Jellyseerr, choose strong admin passwords when you set them up.

- **Traefik Dashboard:** By default, the Traefik dashboard is **not exposed** in our setup (we haven’t published Traefik’s port `8080` or set up a route to it). If you want to access the Traefik dashboard, you have two options:

- Temporarily run `docker compose exec traefik traefik dashboard --port=8080` and connect via SSH port forwarding (advanced), or

- Modify `compose.proxy.yml` to enable the dashboard (add `- "--api.dashboard=true"` and labels for a router with service api@internal). Be sure to secure it with at least basic auth if you expose it, as the Traefik dashboard can control routing.

- **Backups:** It’s wise to routinely back up your configuration and metadata (the `config` directory). This includes Sonarr/Radarr database (which tracks what’s monitored), Jellyfin database (users, watched status), etc. These are all in SQLite or similar files in the config directories. You can simply stop the stack (`docker compose down`) and copy the `/srv/media-stack/config` folder. For live backups without stopping, consider using tools like `duplicity` or snapshotting if your filesystem supports it. Having a backup will save you if an update or mishap corrupts a database.

### 9. Offline Mode and Updates
The stack will function without an active internet connection **except** for the parts that obviously require internet:

Sonarr/Radarr won’t be able to fetch RSS feeds or grab new releases if offline, but their web UI and existing library management will still work on LAN.

Jellyfin will continue to serve already indexed media over LAN without internet (no problem, as long as your clients are on the same network).

Prometheus and Grafana will continue to show local metrics (no internet needed).

If the internet is down, qBittorrent will just pause downloads until connectivity returns. Gluetun might mark unhealthy if it can’t reach the VPN server; it will continually retry connecting. When the internet comes back, it should reconnect automatically. Autoheal may restart Gluetun if it stays unhealthy too long. This self-healing ensures that when connectivity returns, the VPN is re-established fresh.

Watchtower will not be able to check for image updates without internet. It will simply fail to retrieve updates (and likely retry later).

Once internet is restored, all services should resume normal operation automatically.

### 10. Maintenance and Troubleshooting
Viewing Logs: You can view logs for any container to troubleshoot issues:
```
docker compose logs -f sonarr   # Replace 'sonarr' with the service name
```

This can be useful for diagnosing problems (for example, if Sonarr isn’t connecting to qBittorrent, Sonarr logs or qBittorrent logs may show the issue).

**Updating the Stack:** The compose files are modular, so if you add or remove services, update the `COMPOSE_FILE` in `.env` accordingly. For example, if you decide not to run Jellyseerr, you can remove its section from `compose.media.yml`. Or if you want to add another service (say, a Usenet downloader like SABnzbd), you can create a new compose file or add to an existing one and include it. After any changes, run docker compose up -d to apply.

**Resource Usage:** This stack can be heavy. Monitor your CPU/RAM with Grafana. If Jellyfin is consuming too much CPU (transcoding), consider upgrading hardware or limiting its CPU usage via Docker (compose supports `deploy.resources` limits for memory and CPU – you could add those if needed). The Prometheus + Grafana combo also uses some memory and CPU. If you have a very low-power device, you might disable monitoring or reduce scrape frequency to lighten the load.

**Container Restart Order:** Normally, Compose will start containers in an order that respects dependencies. We have set some `depends_on` (for example, *Arrs depend on Gluetun being up for qBittorrent). If you ever reboot your system, Docker should restart all containers; however, there’s a slight chance that, say, Sonarr starts before Gluetun is fully ready. In such a case, Sonarr might mark qBittorrent as unavailable until it can reconnect. Usually it will retry automatically. If not, a quick `docker compose restart sonarr` would fix it. We have tried to mitigate this by using healthchecks and dependencies where appropriate.

### 11. Optional: Using Profiles to Select Components
By default, the `.env` file aggregates all compose files, so `docker compose up -d` brings up everything. Docker Compose also supports profiles which allow selective startup of services. In this configuration, we did not explicitly set up profiles (to keep it straightforward). If you want to only run a subset of services (for example, you might not want the monitoring stack running all the time), you can:

Remove or comment out the undesired compose file in the `COMPOSE_FILE` variable (e.g., omit `compose.monitoring.yml`).

Or define profiles in the compose YAML and activate/deactivate them via environment or command-line. (This is advanced usage; see Docker Compose documentation on profiles.)

### 12. Shutting Down and Starting on Boot
To stop the entire stack, you can run:
```
docker compose down
```

This will stop and remove the containers (but not delete any volumes or config files on the host). To start it up again, use the `up -d` command as before. If you want the stack to start on system boot, there are a few approaches:

Use Docker’s built-in restart policy. We have set most services with `restart: unless-stopped`, which means Docker will try to keep them running. If the Docker daemon starts on boot, it typically will restart containers with this policy.

If you find services are not starting automatically after a reboot, consider using systemd to run `docker compose up -d` on startup, or enable the Docker service’s auto-start features for containers.

### 13. License
This project’s code (the compose files, scripts, etc.) is licensed under the MIT License (see LICENSE). This means you’re free to use and modify it as you like. Please note, this does not apply to the Docker images used – each image has its own licensing (most are open source).

Happy self-hosting! If you run into any issues, feel free to open an issue on the GitHub repository. This stack aims to simplify deploying a comprehensive media server, but there are many moving parts. With everything configured correctly, you’ll have an automated system that finds, downloads, organizes, and serves your media collection with minimal manual effort. Enjoy your new media setup!
