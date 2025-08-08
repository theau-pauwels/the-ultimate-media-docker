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

qBittorrent’s web UI is not directly exposed on the host network. It’s only reachable through Traefik via the gluetun container. We configured Traefik to route qbittorrent.<DOMAIN> (and <DOMAIN>/qbittorrent) to qBittorrent’s interface through the VPN container.
If the VPN connection is down (or Gluetun is unhealthy), qBittorrent will have no internet connectivity. This is by design (kill-switch). The web UI might still load, but no downloads will happen until the VPN is back up.
Sonarr/Radarr connect to qBittorrent internally. In Sonarr’s download client settings, use gluetun as the host and 8080 as port (since Sonarr and Gluetun are on the same Docker network, Sonarr can reach qBittorrent’s interface via the Gluetun container’s hostname). We set a network alias “qbittorrent” on the Gluetun service for convenience, so you can alternatively use qbittorrent as the host name. The username/password are what you set in WEBUI_USER/WEBUI_PASS.
You do not need to open qBittorrent’s port on your router or host – incoming torrent connections are handled via the VPN’s port forwarding (if your VPN provider supports it). Gluetun can manage VPN port forwarding (for providers like PIA) automatically. If using a custom WireGuard server or a provider that doesn’t support port forwarding, you might not get incoming connections – which can slow torrenting. In that case, consider enabling DHT and using well-seeded torrents or switching to a provider that supports port forwarding.
E. Media Import and Hardlinks: This setup separates download locations and final media locations. Sonarr and Radarr will move files from the downloads directories to the media library after download (and optionally delete or hardlink). By default, the volumes for /downloads and /movies//tv in the container are separate mounts, which means hardlinks will not work (the files will be copied instead of hardlinked). If you prefer to use hardlinks to save space (i.e., not duplicate data when importing), you need to ensure the downloads and media directories are on the same filesystem mount. Since we have them under the same base /srv/media-stack, if that is a single filesystem, it actually can support hardlinks – however, Docker’s separate volume mount may still treat them as distinct. To use hardlinks:
One approach is to modify the compose files to mount a common parent directory for both downloads and media in Sonarr/Radarr (so they see both paths under one volume). For simplicity, our default config does not do this. Instead, we recommend disabling hardlinks in Sonarr/Radarr: In Sonarr/Radarr settings, go to Media Management and uncheck “Use Hardlinks instead of Copy”. This will ensure Sonarr/Radarr will copy and then delete original files.
If you wish to enable hardlinking, you could mount /srv/media-stack as /data in the containers and have subfolders (e.g. /data/media/movies and /data/downloads/movies). This requires adjusting the volume mounts. (Advanced users only – the current setup is perfectly functional with copies.)
F. Monitoring Dashboards: Once Prometheus and Grafana are running, you can import premade dashboards to visualize your system:
Node Exporter (Host metrics): Import Prometheus’ official Node Exporter dashboard (available on Grafana’s Dashboards site) to see CPU, memory, disk, etc.
cAdvisor (Docker container metrics): There are community dashboards that show per-container CPU/RAM usage, I/O, network, etc. You should be able to find one on Grafana’s dashboard repository (for example, search for “Docker Metrics by cAdvisor”).
Traefik metrics: By default, we haven’t enabled Traefik metrics, but you can turn them on in Traefik’s config if desired and then import a Traefik dashboard.
G. Jellyfin Setup: In Jellyfin’s dashboard, you may want to set up hardware acceleration for transcoding (if your server has a GPU and you use it). This usually involves passing through the GPU (e.g., via Docker device parameters which are not included by default in our compose). If you plan to do a lot of transcoding, research how to enable NVIDIA/Intel QuickSync in Jellyfin’s container. (This will require installing drivers on the host and modifying the compose file to add the device). For many users who stream in original format on a LAN, this might not be necessary.
10. Security Considerations
This stack is designed with security in mind, but you are responsible for the secure operation of your server. Keep these points in mind:
VPN (Privacy for Downloads): qBittorrent’s traffic is entirely encapsulated in the VPN tunnel. This protects your peer-to-peer traffic from prying eyes. Ensure your VPN credentials/config remain secure. If the VPN disconnects, qBittorrent cannot reach the internet (so no accidental leaking of your IP). Gluetun’s healthcheck will mark it unhealthy if the VPN is down; Autoheal will attempt to restart it if that happens. If you see frequent restarts, investigate your VPN stability.
Firewall: Even though qBittorrent’s web UI isn’t exposed directly, other services (Sonarr, Radarr, etc.) might be accessible through Traefik. If you’re accessing the services over the internet, consider restricting ports 80/443 on your router or using a VPN (the other kind – e.g., WireGuard or Tailscale for yourself) to access them, or implement authentication:
You can add HTTP Basic Auth in Traefik for sensitive apps (Traefik dashboard, Sonarr, etc.) by adding middleware in Traefik’s config. For example, protect Traefik’s own dashboard or even the *Arr apps with a password if you want an extra layer (since by default *Arr have no auth). Basic Auth middleware can be configured via labels on the service in the compose file or via a dynamic config file.
Another approach is to keep these services accessible only on your LAN (don’t expose ports 80/443 to the internet) and use a VPN or SSH tunnel to access your home network remotely.
Auto-Updates (Watchtower): Watchtower will automatically pull the latest Docker images for all containers and restart them with the existing configuration. This keeps you up-to-date with security patches and new features. However, automatic updates can occasionally break things if a new version changes something. It’s a good practice to monitor your containers (via logs or Grafana) so you notice if something goes wrong after an update. If you prefer to control updates manually, you can disable Watchtower by removing the service from compose.utils.yml or by stopping its container.
Autoheal: Autoheal will restart containers that become unhealthy (as determined by their Docker healthchecks). For example, if the VPN container is unhealthy or if Jellyfin’s healthcheck fails (not configured by default to have one), they’ll restart. This helps keep your stack running smoothly without manual intervention. However, it’s not a substitute for investigating root causes. Frequent unhealthy restarts could indicate a problem (e.g., an unreliable VPN server or a misconfiguration).
Admin Interfaces: As mentioned, change all default passwords. Don’t leave the Grafana admin with the default password, and similarly for qBittorrent. For Jellyfin/Jellyseerr, choose strong admin passwords when you set them up.
Traefik Dashboard: By default, the Traefik dashboard is not exposed in our setup (we haven’t published Traefik’s port 8080 or set up a route to it). If you want to access the Traefik dashboard, you have two options:
Temporarily run docker compose exec traefik traefik dashboard --port=8080 and connect via SSH port forwarding (advanced), or
Modify compose.proxy.yml to enable the dashboard (add - "--api.dashboard=true" and labels for a router with service api@internal). Be sure to secure it with at least basic auth if you expose it, as the Traefik dashboard can control routing.
Backups: It’s wise to routinely back up your configuration and metadata (the config directory). This includes Sonarr/Radarr database (which tracks what’s monitored), Jellyfin database (users, watched status), etc. These are all in SQLite or similar files in the config directories. You can simply stop the stack (docker compose down) and copy the /srv/media-stack/config folder. For live backups without stopping, consider using tools like duplicity or snapshotting if your filesystem supports it. Having a backup will save you if an update or mishap corrupts a database.
11. Offline Mode and Updates
The stack will function without an active internet connection except for the parts that obviously require internet:
Sonarr/Radarr won’t be able to fetch RSS feeds or grab new releases if offline, but their web UI and existing library management will still work on LAN.
Jellyfin will continue to serve already indexed media over LAN without internet (no problem, as long as your clients are on the same network).
Prometheus and Grafana will continue to show local metrics (no internet needed).
If the internet is down, qBittorrent will just pause downloads until connectivity returns. Gluetun might mark unhealthy if it can’t reach the VPN server; it will continually retry connecting. When the internet comes back, it should reconnect automatically. Autoheal may restart Gluetun if it stays unhealthy too long. This self-healing ensures that when connectivity returns, the VPN is re-established fresh.
Watchtower will not be able to check for image updates without internet. It will simply fail to retrieve updates (and likely retry later).
Once internet is restored, all services should resume normal operation automatically.
12. Maintenance and Troubleshooting
Viewing Logs: You can view logs for any container to troubleshoot issues:
docker compose logs -f sonarr   # Replace 'sonarr' with the service name
This can be useful for diagnosing problems (for example, if Sonarr isn’t connecting to qBittorrent, Sonarr logs or qBittorrent logs may show the issue).
Updating the Stack: The compose files are modular, so if you add or remove services, update the COMPOSE_FILE in .env accordingly. For example, if you decide not to run Jellyseerr, you can remove its section from compose.media.yml. Or if you want to add another service (say, a Usenet downloader like SABnzbd), you can create a new compose file or add to an existing one and include it. After any changes, run docker compose up -d to apply.
Resource Usage: This stack can be heavy. Monitor your CPU/RAM with Grafana. If Jellyfin is consuming too much CPU (transcoding), consider upgrading hardware or limiting its CPU usage via Docker (compose supports deploy.resources limits for memory and CPU – you could add those if needed). The Prometheus + Grafana combo also uses some memory and CPU. If you have a very low-power device, you might disable monitoring or reduce scrape frequency to lighten the load.
Container Restart Order: Normally, Compose will start containers in an order that respects dependencies. We have set some depends_on (for example, *Arrs depend on Gluetun being up for qBittorrent). If you ever reboot your system, Docker should restart all containers; however, there’s a slight chance that, say, Sonarr starts before Gluetun is fully ready. In such a case, Sonarr might mark qBittorrent as unavailable until it can reconnect. Usually it will retry automatically. If not, a quick docker compose restart sonarr would fix it. We have tried to mitigate this by using healthchecks and dependencies where appropriate.
13. Optional: Using Profiles to Select Components
By default, the .env file aggregates all compose files, so docker compose up -d brings up everything. Docker Compose also supports profiles which allow selective startup of services. In this configuration, we did not explicitly set up profiles (to keep it straightforward). If you want to only run a subset of services (for example, you might not want the monitoring stack running all the time), you can:
Remove or comment out the undesired compose file in the COMPOSE_FILE variable (e.g., omit compose.monitoring.yml).
Or define profiles in the compose YAML and activate/deactivate them via environment or command-line. (This is advanced usage; see Docker Compose documentation on profiles.)
14. Shutting Down and Starting on Boot
To stop the entire stack, you can run:
docker compose down
This will stop and remove the containers (but not delete any volumes or config files on the host). To start it up again, use the up -d command as before. If you want the stack to start on system boot, there are a few approaches:
Use Docker’s built-in restart policy. We have set most services with restart: unless-stopped, which means Docker will try to keep them running. If the Docker daemon starts on boot, it typically will restart containers with this policy.
If you find services are not starting automatically after a reboot, consider using systemd to run docker compose up -d on startup, or enable the Docker service’s auto-start features for containers.
15. License
This project’s code (the compose files, scripts, etc.) is licensed under the MIT License (see LICENSE). This means you’re free to use and modify it as you like. Please note, this does not apply to the Docker images used – each image has its own licensing (most are open source).
Happy self-hosting! If you run into any issues, feel free to open an issue on the GitHub repository. This stack aims to simplify deploying a comprehensive media server, but there are many moving parts. With everything configured correctly, you’ll have an automated system that finds, downloads, organizes, and serves your media collection with minimal manual effort. Enjoy your new media setup!
Configuration Files
Below are all the configuration and compose files for this project. Each is fully annotated where appropriate:
docker-compose.yml (Aggregator)
This primary compose file aggregates the others and defines the common network used by all services.
version: "3.8"
# Main aggregator compose file - includes all other compose files via .env COMPOSE_FILE
networks:
  media-net:
    name: media-stack
    driver: bridge
    # This network is shared by all services for inter-communication
    # (Traefik uses this network to reach other containers, etc.)
services:
  # (No primary services defined in this file; see compose.*.yml files for actual services)
compose.proxy.yml (Traefik reverse proxy)
This compose file defines the Traefik service which acts as a reverse proxy and entrypoint for all web traffic. It listens on ports 80 and 443, obtains SSL certificates, and routes requests to the appropriate service based on domain/path.
version: "3.8"
services:
  traefik:
    image: traefik:latest
    container_name: traefik
    restart: unless-stopped
    networks:
      - media-net
    ports:
      - "80:80"     # HTTP
      - "443:443"   # HTTPS
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro   # Allows Traefik to monitor Docker events (for auto-config)
      - ${BASE_DIR}/config/traefik/acme.json:/etc/traefik/acme.json   # Persist ACME (LetsEncrypt) certificates
    command:
      - "--providers.docker=true"
      - "--providers.docker.network=media-stack"    # Monitor containers on the "media-stack" network
      - "--providers.docker.exposedbydefault=false" # Only expose containers that have explicit labels
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"   # Redirect HTTP to HTTPS
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--entrypoints.websecure.http.tls=true"
      - "--certificatesresolvers.le.acme.tlschallenge=true"   # Use TLS-ALPN challenge for LetsEncrypt
      - "--certificatesresolvers.le.acme.email=${LETSENCRYPT_EMAIL}"   # Email for LetsEncrypt notifications
      - "--certificatesresolvers.le.acme.storage=/etc/traefik/acme.json"
    labels:
      # (Optional) Example: route Traefik's own dashboard (disabled by default for security)
      # If needed, uncomment and set up basic auth:
      # - "traefik.http.routers.traefik.rule=Host(`traefik.${DOMAIN}`)"
      # - "traefik.http.routers.traefik.service=api@internal"
      # - "traefik.http.routers.traefik.entrypoints=websecure"
      # - "traefik.http.routers.traefik.tls.certresolver=le"
      # - "traefik.http.routers.traefik.middlewares=auth@file"
      # (Where auth@file is a basic auth middleware defined in a file, not configured here by default)
    depends_on:
      - gluetun  # Ensure VPN is up (and thus network ready) before starting Traefik
    # Note: Traefik doesn't necessarily need to depend on gluetun, but forcing an order can help avoid
    # any networking race conditions.
Traefik Notes:
It uses the Docker provider to dynamically discover containers with labels and create routes. We set exposedbydefault=false to ensure that only containers we explicitly label will be exposed.
The certificatesresolvers.le configuration enables Let’s Encrypt. It uses the TLS challenge, so port 443 must be accessible from the internet. Certificates and keys will be stored in acme.json.
HTTP (port 80) is redirected to HTTPS (port 443). If you want to allow plain HTTP (not recommended for external traffic), you can remove the redirect commands.
The example labels for Traefik’s own dashboard are commented out for security. If you want to enable the dashboard, it’s strongly advised to also set up authentication (Traefik supports basic auth via a middleware).
compose.arr.yml (Sonarr, Radarr, Lidarr, Prowlarr, Bazarr)
This compose file defines the *Arr suite of services for managing media.
version: "3.8"
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${BASE_DIR}/config/sonarr:/config
      - ${BASE_DIR}/media/tv:/tv       # TV library path (Sonarr will import shows here)
      - ${BASE_DIR}/downloads/tv:/downloads  # Downloaded episodes from qBittorrent
    labels:
      - "traefik.enable=true"
      # Router for subdomain access (e.g., sonarr.example.com)
      - "traefik.http.routers.sonarr.rule=Host(`sonarr.${DOMAIN}`)"
      - "traefik.http.routers.sonarr.entrypoints=websecure"
      - "traefik.http.routers.sonarr.tls.certresolver=le"
      # Router for path-based access (e.g., example.com/sonarr)
      - "traefik.http.routers.sonarr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/sonarr`)"
      - "traefik.http.routers.sonarr-path.entrypoints=websecure"
      - "traefik.http.routers.sonarr-path.tls.certresolver=le"
      - "traefik.http.routers.sonarr-path.service=sonarr"   # Use the same service as main
      - "traefik.http.services.sonarr.loadbalancer.server.port=8989"
      # Note: We recommend setting Sonarr's internal URL Base to /sonarr if using path routing.

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${BASE_DIR}/config/radarr:/config
      - ${BASE_DIR}/media/movies:/movies   # Movie library path
      - ${BASE_DIR}/downloads/movies:/downloads  # Downloaded movies
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.radarr.rule=Host(`radarr.${DOMAIN}`)"
      - "traefik.http.routers.radarr.entrypoints=websecure"
      - "traefik.http.routers.radarr.tls.certresolver=le"
      - "traefik.http.routers.radarr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/radarr`)"
      - "traefik.http.routers.radarr-path.entrypoints=websecure"
      - "traefik.http.routers.radarr-path.tls.certresolver=le"
      - "traefik.http.routers.radarr-path.service=radarr"
      - "traefik.http.services.radarr.loadbalancer.server.port=7878"
      # Radarr's internal port 7878 is used.
      # Set Radarr's URL Base to /radarr if using path.

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${BASE_DIR}/config/lidarr:/config
      - ${BASE_DIR}/media/music:/music    # Music library path
      - ${BASE_DIR}/downloads/music:/downloads  # Downloaded music
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.lidarr.rule=Host(`lidarr.${DOMAIN}`)"
      - "traefik.http.routers.lidarr.entrypoints=websecure"
      - "traefik.http.routers.lidarr.tls.certresolver=le"
      - "traefik.http.routers.lidarr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/lidarr`)"
      - "traefik.http.routers.lidarr-path.entrypoints=websecure"
      - "traefik.http.routers.lidarr-path.tls.certresolver=le"
      - "traefik.http.routers.lidarr-path.service=lidarr"
      - "traefik.http.services.lidarr.loadbalancer.server.port=8686"
      # Lidarr uses port 8686 internally.
      # Set Lidarr's URL Base to /lidarr if using path.

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${BASE_DIR}/config/prowlarr:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prowlarr.rule=Host(`prowlarr.${DOMAIN}`)"
      - "traefik.http.routers.prowlarr.entrypoints=websecure"
      - "traefik.http.routers.prowlarr.tls.certresolver=le"
      - "traefik.http.routers.prowlarr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/prowlarr`)"
      - "traefik.http.routers.prowlarr-path.entrypoints=websecure"
      - "traefik.http.routers.prowlarr-path.tls.certresolver=le"
      - "traefik.http.routers.prowlarr-path.service=prowlarr"
      - "traefik.http.services.prowlarr.loadbalancer.server.port=9696"
      # Prowlarr on port 9696. URL Base to /prowlarr if using path.

  bazarr:
    image: lscr.io/linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ${BASE_DIR}/config/bazarr:/config
      - ${BASE_DIR}/media/movies:/movies
      - ${BASE_DIR}/media/tv:/tv
      # Bazarr needs access to media files to scan and add subtitles.
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.bazarr.rule=Host(`bazarr.${DOMAIN}`)"
      - "traefik.http.routers.bazarr.entrypoints=websecure"
      - "traefik.http.routers.bazarr.tls.certresolver=le"
      - "traefik.http.routers.bazarr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/bazarr`)"
      - "traefik.http.routers.bazarr-path.entrypoints=websecure"
      - "traefik.http.routers.bazarr-path.tls.certresolver=le"
      - "traefik.http.routers.bazarr-path.service=bazarr"
      - "traefik.http.services.bazarr.loadbalancer.server.port=6767"
      # Bazarr uses port 6767. Set Base URL to /bazarr if using path.
Arr Services Notes:
All *Arr containers are configured with PUID/PGID for permissions and timezone.
Each has volumes for config and for media/downloads. The downloads directories (/downloads) are where qBittorrent will put completed files. We separated them by category (movies vs tv vs music) for organization. Ensure you configure qBittorrent to use these subfolders, or simply have Sonarr/Radarr move from a common /downloads root.
Traefik labels: We define two routers for each service: one matching a subdomain, one matching a path prefix. Both routers point to the same service (the container), and we specify the internal service port for Traefik. This way, the service is accessible at e.g. sonarr.my-domain.com and also my-domain.com/sonarr. You can use either or both. If you only use subdomains, you can ignore the path-based addresses (they will still work, but the apps might not function correctly unless base URLs are set, as discussed).
The depends_on: gluetun for Traefik ensures Traefik starts after the VPN container is up. We did not put depends_on for each *Arr on qBittorrent/Gluetun because they don’t strictly need to wait (they’ll retry connections to qBittorrent). But you can add if desired.
compose.media.yml (Jellyfin and Jellyseerr)
This compose file defines the media server and its companion request app.
version: "3.8"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      # Hardware acceleration options (optional):
      # - NVIDIA_VISIBLE_DEVICES=all
      # - NVIDIA_DRIVER_CAPABILITIES=video,compute,utility
    volumes:
      - ${BASE_DIR}/config/jellyfin:/config
      - ${BASE_DIR}/media:/media    # Mount entire media library (Jellyfin libraries will be subfolders)
      - ${BASE_DIR}/media:/data     # (Jellyfin can use /data as alternate, but not strictly needed)
      # - /path/to/host/transcode:/transcode  # Optional: for transcoding temp if needed
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.${DOMAIN}`)"
      - "traefik.http.routers.jellyfin.entrypoints=websecure"
      - "traefik.http.routers.jellyfin.tls.certresolver=le"
      - "traefik.http.routers.jellyfin-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/jellyfin`)"
      - "traefik.http.routers.jellyfin-path.entrypoints=websecure"
      - "traefik.http.routers.jellyfin-path.tls.certresolver=le"
      - "traefik.http.routers.jellyfin-path.service=jellyfin"
      - "traefik.http.services.jellyfin.loadbalancer.server.port=8096"
      # Jellyfin default HTTP port is 8096. (HTTPS would be 8920 if enabled internally, but we'll use Traefik's TLS)
      # Note: Set Jellyfin Base URL to /jellyfin if using path-based routing.
    ports:
      - "8096:8096/tcp"   # (Optional) expose internally on LAN without Traefik, e.g., for debugging or DLNA
      - "1900:1900/udp"   # (Optional) DLNA discovery
      - "7359:7359/udp"   # (Optional) DLNA
    # If you don't need DLNA or direct LAN access, you can omit the above ports. Traefik will handle HTTP(S).

  jellyseerr:
    image: fallenbagel/jellyseerr:latest
    container_name: jellyseerr
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - LOG_LEVEL=info
      - TZ=${TZ}
      # Jellyseerr uses a built-in SQLite by default for its DB (stored under /config).
      # If connecting to Jellyfin, you'll configure that via the web UI on first run.
    volumes:
      - ${BASE_DIR}/config/jellyseerr:/app/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyseerr.rule=Host(`jellyseerr.${DOMAIN}`)"
      - "traefik.http.routers.jellyseerr.entrypoints=websecure"
      - "traefik.http.routers.jellyseerr.tls.certresolver=le"
      - "traefik.http.routers.jellyseerr-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/jellyseerr`)"
      - "traefik.http.routers.jellyseerr-path.entrypoints=websecure"
      - "traefik.http.routers.jellyseerr-path.tls.certresolver=le"
      - "traefik.http.routers.jellyseerr-path.service=jellyseerr"
      - "traefik.http.services.jellyseerr.loadbalancer.server.port=5055"
      # Jellyseerr default port is 5055.
      # Note: You'll typically access Jellyseerr via browser and configure it to point to Jellyfin/Sonarr/Radarr.
Media Services Notes:
Jellyfin: We mounted the entire /media directory. In Jellyfin’s web UI, you will add libraries and point them to e.g. /media/movies, /media/tv, etc. We commented out hardware acceleration env variables; if you have an NVIDIA GPU and have installed NVIDIA Docker runtime, you can uncomment and adjust accordingly. The optional port mappings allow you to use Jellyfin on your LAN (e.g., via http://<serverIP>:8096) or to enable DLNA for local devices. If you do not need DLNA or direct access, you can remove the ports section — Traefik will still allow access via domain.
Jellyseerr: This is a fork of Overseerr for Jellyfin integration. On first launch, you’ll go through the configuration to connect it to Jellyfin (you’ll need Jellyfin’s host/port and an API key from Jellyfin) and to Sonarr/Radarr (for automatic adding of requests). Jellyseerr stores its config/db under config/jellyseerr. There is no authentication configured via Traefik — Jellyseerr itself will have user accounts (you can invite users or allow self-signup depending on your settings). If you expose Jellyseerr publicly, consider enabling some sort of captcha or approval for new users if it’s a multi-user environment.
Path-based routing for Jellyfin can be tricky (Jellyfin’s internal paths need the base URL set). For Jellyseerr, path routing should generally work after configuring its External URL setting to include the subpath.
compose.dl.yml (VPN and qBittorrent)
This compose file includes the VPN client (Gluetun) and the qBittorrent downloader that routes through it.
version: "3.8"
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    restart: unless-stopped
    networks:
      - media-net
    cap_add:
      - NET_ADMIN        # Required to change network interfaces
    devices:
      - /dev/net/tun:/dev/net/tun  # TUN device for VPN
    volumes:
      - ${BASE_DIR}/vpn/wg0.conf:/gluetun/wireguard/wg0.conf
      - ${BASE_DIR}/config/qbittorrent:/gluetun/qbittorrent  # Share config directory with qBittorrent if needed
    environment:
      - VPN_SERVICE_PROVIDER=custom
      - VPN_TYPE=wireguard
      - TZ=${TZ}
      - PUID=${PUID}
      - PGID=${PGID}
      # Additional Gluetun settings (optional):
      # - FIREWALL_VPN_INPUT_PORTS=xxxx (if you want to allow incoming port, e.g., for qBittorrent port if forwarded)
      # - TRAEFIK_PROTOCOL=http (gluetun's built-in HTTP proxy can be enabled if needed)
    labels:
      - "traefik.enable=true"
      # Traefik routes for qBittorrent are attached to Gluetun, since qBittorrent's network is the same as Gluetun.
      - "traefik.http.routers.qbittorrent.rule=Host(`qbittorrent.${DOMAIN}`)"
      - "traefik.http.routers.qbittorrent.entrypoints=websecure"
      - "traefik.http.routers.qbittorrent.tls.certresolver=le"
      - "traefik.http.routers.qbittorrent-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/qbittorrent`)"
      - "traefik.http.routers.qbittorrent-path.entrypoints=websecure"
      - "traefik.http.routers.qbittorrent-path.tls.certresolver=le"
      - "traefik.http.routers.qbittorrent-path.service=qbittorrent"
      - "traefik.http.services.qbittorrent.loadbalancer.server.port=8080"
      # We treat Gluetun as the service endpoint for qBittorrent's web UI (on port 8080).
    networks:
      media-net:
        aliases:
          - qbittorrent    # So other containers can resolve 'qbittorrent' to this service (for API access)
    healthcheck:
      test: ["CMD-SHELL", "wget -q --timeout=5 --tries=1 https://api.ipify.org -O /dev/null"]
      interval: 1m
      timeout: 10s
      retries: 3
      start_period: 40s
    # The above healthcheck tries to fetch an external IP. If it fails, gluetun is considered unhealthy.
    # Autoheal will then restart it. You can adjust the interval or disable if it's too sensitive.

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    # Notice no networks here, we use gluetun's network stack
    network_mode: service:gluetun   # All network traffic for qBittorrent flows through Gluetun
    depends_on:
      - gluetun   # Ensure VPN is up before qBittorrent starts
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
      # Credentials are provided via env file for security:
    env_file:
      - ./secrets/admin_credentials.env
      # This file provides WEBUI_USER and WEBUI_PASS
    volumes:
      - ${BASE_DIR}/config/qbittorrent:/config
      - ${BASE_DIR}/downloads:/downloads
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/v2/app/version"]
      interval: 1m
      timeout: 10s
      retries: 3
      start_period: 20s
    # qBittorrent's healthcheck tries to hit its local API. If the VPN is down, this might still return healthy (since it's local).
    # But if qBittorrent crashes or web UI hangs, this can detect it.
Downloader Notes:
Gluetun (VPN): We mount the wg0.conf into the container’s expected location for WireGuard config. We set VPN_SERVICE_PROVIDER=custom and VPN_TYPE=wireguard, which tells Gluetun to use the provided WireGuard config. Ensure wg0.conf is valid; Gluetun will log connection status. We gave Gluetun NET_ADMIN capability and the TUN device for VPN. It joins the media-net so it can talk to Traefik and *Arrs (for the web UI and API calls respectively). We set a healthcheck using a simple external request to detect connectivity loss. The interval is 60s; if your VPN is unstable, you might see restarts. You can disable or modify this healthcheck by commenting it out or changing the command (for instance, gluetun has an internal healthcheck command; we used a generic approach).
The label routing for qBittorrent is applied to Gluetun. Because qbittorrent service has network_mode: service:gluetun, it doesn’t have its own network presence for Traefik to latch onto. Instead, we attach Gluetun to the Traefik network and give it the labels. We also give Gluetun a network alias “qbittorrent” so other containers (Sonarr/Radarr) can access qBittorrent’s web UI/API by using http://qbittorrent:8080.
The traefik.http.services.qbittorrent.loadbalancer.server.port=8080 tells Traefik to forward to port 8080 on the Gluetun container (which is effectively qBittorrent’s UI).
No ports are exposed on the host for qBittorrent, to avoid any leaks; all access is via Traefik. Gluetun does expose a built-in DNS and possibly other services on its own network, but on the media-net those aren’t used. We do not expose Gluetun’s ports on the host either.
If your VPN provider supports port forwarding and you need it (to improve torrent performance), you can configure Gluetun to get a port. For example, with PIA you’d set PORT_FORWARDING=on and possibly specify region. Gluetun will output the forwarded port. You then need to tell qBittorrent to use that port for incoming connections (by default, qBittorrent might randomize or use 8999, etc.). You can set qBittorrent’s listening port via an env var or config file. For simplicity, we haven’t included that. Many VPN providers don’t support port forwarding; even without, you can still download, albeit potentially with fewer peers.
qBittorrent: It uses LinuxServer’s image. We configure it to run in Gluetun’s network namespace. That means qBittorrent has no separate network interface from Gluetun. It shares Gluetun’s IP and network stack. This ensures all its traffic goes through the VPN tunnel. We mount the downloads directory and its config. The WEBUI_PORT is set to 8080 (the default in LSIO image), and we provide WEBUI_USER and WEBUI_PASS via the admin_credentials.env file. On first run, if those are set, it will use them. If not provided, the container will generate a random password (and log it) for security (LSIO changed this from default admin/adminadmin).
The healthcheck for qBittorrent hits the internal API endpoint to verify it’s responding. If it fails (e.g., qBittorrent crashed), Autoheal would restart it.
Sonarr/Radarr should be configured to connect to qBittorrent at http://gluetun:8080 or http://qbittorrent:8080 (since we set that alias). Use the credentials from admin_credentials.env. The *Arrs are on the same media-net network, so they can resolve those hostnames.
We did not publish qBittorrent’s ports (like 6881) to the host. Torrent traffic goes through the VPN tunnel directly. If you check qBittorrent’s settings, you’ll see a listening port (e.g., 6881). This is bound inside the container’s network (which is the VPN). If your VPN assigns a different external port via port forwarding, you might need to update qBittorrent’s settings to match that for incoming connections. This can be done in qBittorrent’s UI or by environment variable (LSIO doesn’t document one for that, but you can set in config).
The volume for qBittorrent’s config is shared with Gluetun (mounted under /gluetun/qbittorrent). This isn’t strictly required; it’s a just-in-case if Gluetun needed to check something or if you wanted to use Gluetun’s built-in TinyProxy or other features. You can remove that shared mount if not needed. It does no harm though.
compose.monitoring.yml (Prometheus, Grafana, Node Exporter, cAdvisor)
This compose file sets up the monitoring stack.
version: "3.8"
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    networks:
      - media-net
    volumes:
      - ${BASE_DIR}/config/prometheus:/etc/prometheus       # for prometheus.yml config
      - ${BASE_DIR}/monitoring/prometheus:/prometheus       # data storage
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.console.libraries=/usr/share/prometheus/console_libraries"
      - "--web.console.templates=/usr/share/prometheus/consoles"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prometheus.rule=Host(`prometheus.${DOMAIN}`)"
      - "traefik.http.routers.prometheus.entrypoints=websecure"
      - "traefik.http.routers.prometheus.tls.certresolver=le"
      - "traefik.http.routers.prometheus-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/prometheus`)"
      - "traefik.http.routers.prometheus-path.entrypoints=websecure"
      - "traefik.http.routers.prometheus-path.tls.certresolver=le"
      - "traefik.http.routers.prometheus-path.service=prometheus"
      - "traefik.http.services.prometheus.loadbalancer.server.port=9090"
      # Prometheus UI on 9090. (Note: Prometheus has no auth - consider not exposing publicly or secure via Traefik basic auth.)
    depends_on:
      - nodeexporter
      - cadvisor

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - GF_SECURITY_ADMIN_USER=${GF_SECURITY_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}
      - GF_SECURITY_DISABLE_GRAVATAR=true
      - GF_USERS_ALLOW_SIGN_UP=false
      # Above: disable user signup, set admin user/password from secrets.
      # (These come from admin_credentials.env via .env variable expansion or you can set directly here.)
      - TZ=${TZ}
    env_file:
      - ./secrets/admin_credentials.env   # Contains GF_SECURITY_ADMIN_USER/PASSWORD as well
    volumes:
      - ${BASE_DIR}/config/grafana:/etc/grafana   # for custom configs or provisioning (optional)
      - ${BASE_DIR}/monitoring/grafana:/var/lib/grafana  # persistent Grafana data (sqlite DB, etc.)
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.rule=Host(`grafana.${DOMAIN}`)"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.tls.certresolver=le"
      - "traefik.http.routers.grafana-path.rule=Host(`${DOMAIN}`) && PathPrefix(`/grafana`)"
      - "traefik.http.routers.grafana-path.entrypoints=websecure"
      - "traefik.http.routers.grafana-path.tls.certresolver=le"
      - "traefik.http.routers.grafana-path.service=grafana"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"
      # Grafana default port 3000.

  nodeexporter:
    image: prom/node-exporter:latest
    container_name: nodeexporter
    restart: unless-stopped
    networks:
      - media-net
    # To monitor host metrics, give access to host's proc and sys:
    pid: host
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro
    command:
      - "--path.rootfs=/host"   # Use the host path for reading /proc and /sys
    expose:
      - "9100"
    # No Traefik labels here because we don't need to expose Node Exporter outside.
    # Prometheus will scrape it internally at nodeexporter:9100.

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    networks:
      - media-net
    devices:
      - /dev/kmsg:/dev/kmsg   # might be needed for cAdvisor to read kernel messages
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    expose:
      - "8080"
    # cAdvisor also doesn't need external exposure; Prometheus scrapes it at cadvisor:8080/metrics.
Monitoring Notes:
Prometheus: Uses the official image. We mount a config directory and specify prometheus.yml via command. You should create a basic prometheus.yml in ${BASE_DIR}/config/prometheus before launching. A simple config is provided below in the Prometheus Config section. The compose sets up scrape targets expecting the service names prometheus, nodeexporter, cadvisor (which we use in config). We expose Prometheus via Traefik (optionally, you might not want it public as there’s no auth). At minimum, maybe keep it local or add auth middleware. For now, no auth on it – be cautious exposing it.
Grafana: We use environment to set the admin user and password from the secrets file. We also disable signups and Gravatar for privacy. Grafana is exposed via Traefik. Grafana has its own user management and can be secured with the password we provided. The config/grafana volume can be used if you want to drop in any custom config or provisioning (like predefined dashboards or data sources). We didn’t include a provisioning in this initial setup, so you’ll configure things via the UI.
Node Exporter: Runs in host PID mode and reads host’s /proc and /sys. This exposes host hardware stats. It’s not exposed via Traefik (no need), Prometheus scrapes it. The expose on 9100 makes it accessible within the media-net network. In prometheus.yml, we’ll use nodeexporter:9100 as the target.
cAdvisor: Collects Docker container metrics and exposes them on port 8080 (on /metrics). We expose 8080 internally. Prometheus will scrape cadvisor:8080. cAdvisor requires access to Docker’s sock to list containers and /sys and Docker’s var/lib to gather metrics. It runs unprivileged (except it might need /dev/kmsg).
Prometheus Config: Create a file ${BASE_DIR}/config/prometheus/prometheus.yml with contents like:
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['prometheus:9090']
  - job_name: nodeexporter
    static_configs:
      - targets: ['nodeexporter:9100']
  - job_name: cadvisor
    static_configs:
      - targets: ['cadvisor:8080']
This matches our service names. You can add more jobs (for Traefik metrics or others if needed).
After launching, verify Prometheus can reach the targets (check Status > Targets in Prometheus UI or look at Prometheus logs). Grafana will need to be configured with a Prometheus data source (URL http://prometheus:9090 from Grafana’s perspective on the same network).
Security: Grafana requires login (we set a strong default, change if needed). Prometheus has no login – if you don’t want it public, consider removing Traefik labels for Prometheus or adding basic auth via Traefik. Node Exporter and cAdvisor are not exposed externally.
compose.utils.yml (Autoheal and Watchtower)
Utility services for self-healing and updating.
version: "3.8"
services:
  autoheal:
    image: willfarrell/autoheal:latest
    container_name: autoheal
    restart: unless-stopped
    networks:
      - media-net
    # Autoheal can monitor all Docker containers and restart unhealthy ones
    environment:
      - AUTOHEAL_CONTAINER_LABEL=all   # watch all containers for health (those with healthcheck defined)
      - AUTOHEAL_INTERVAL=30           # check every 30 seconds
      - AUTOHEAL_DEFAULT_STOP_TIMEOUT=20
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    # No labels; we don't need to expose this.

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    networks:
      - media-net
    environment:
      - WATCHTOWER_CLEANUP=true          # remove old images
      - WATCHTOWER_CHECK_INTERVAL=86400  # check once a day (in seconds, 86400s = 24h)
      # - WATCHTOWER_LABEL_ENABLE=true   # (optional) if you prefer to only update containers with a certain label
      # - WATCHTOWER_LIFECYCLE_HOOKS=true # allow pre/post update hooks (not used here)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    # No labels (not exposed). It will update all containers unless configured otherwise.
Utilities Notes:
Autoheal: We set it to monitor all containers (with a healthcheck). It connects to Docker socket to issue restarts. It will only act on containers that have the label autoheal=true or if AUTOHEAL_CONTAINER_LABEL=all, then any container with a failing healthcheck. We chose the all approach. This means if any container goes unhealthy (exits code !=0 for health test) it will restart it after the interval. We gave an interval of 30s. If you find Gluetun flapping (some VPN servers might cause intermittent failures in healthcheck), you might increase the interval or refine the healthcheck. You can also switch to labeling individual containers you want autohealed instead. In this stack: gluetun and qbittorrent have healthchecks defined, as does qbittorrent. Those are the main ones that might trigger autoheal. If, say, Node Exporter or others had a healthcheck and failed, they’d restart too.
Watchtower: We set it to check once every 24h for updates to your images. When it finds an update, it will stop the container and restart it with the new image (pulling it first). We enabled cleanup to remove old image layers. This keeps your system from accumulating unused images.
Caution: With automatic updates, occasionally an image update might introduce changes requiring manual intervention (e.g., config format changes). It’s a good idea to monitor your services after an update window. You can configure Watchtower to send notifications (Slack, email, etc.) via environment variables if you want alerts on updates (not shown here).
If you prefer to only update some containers and not others, you could enable WATCHTOWER_LABEL_ENABLE=true and then add the label com.centurylinklabs.watchtower.enable=true to each service you want updated. Our config currently will update everything including itself. By default, Watchtower will skip restarting itself until the end.
Watchtower uses the Docker socket to manage containers/images. It’s a powerful tool; ensure it’s up to date itself.
Environment Variables (.env file)
Below is the complete .env file content, which centralizes configuration:
# .env - Environment Configuration for the Ultimate Media Docker stack

# Compose File Aggregation: include all modular compose files
COMPOSE_PATH_SEPARATOR=:
COMPOSE_FILE=docker-compose.yml:compose.proxy.yml:compose.arr.yml:compose.media.yml:compose.dl.yml:compose.monitoring.yml:compose.utils.yml

# User and Group for service containers (ensure it matches the owner of /srv/media-stack)
PUID=1000
PGID=1000

# Timezone (use your local TZ for correct scheduling/log times)
TZ=UTC
# Examples: "America/New_York", "Europe/Brussels", etc.

# Domain configuration for Traefik routing
DOMAIN=example.com
# Set this to the base domain you'll use for accessing services.
# If using subdomains: e.g., sonarr.example.com, etc., ensure DNS records exist.
# If using path routing (e.g., example.com/sonarr), still set your main domain here.

# Let's Encrypt email (for certificate notifications)
LETSENCRYPT_EMAIL=your-email@example.com
# Recommended if using HTTPS. Traefik will register your email with Let's Encrypt.
# If you won't use Let's Encrypt/HTTPS, you can leave this blank, but then adjust Traefik settings accordingly.

# Base directory for media stack data on host
BASE_DIR=/srv/media-stack
# All data (configs, media, downloads) will reside under this directory.
# Change if you want to use a different path. Ensure directory exists and is writable by PUID/PGID.

# --- Additional optional settings ---

# (If using Cloudflare DNS challenge for SSL instead of TLS challenge, you could add:)
# CF_DNS_API_TOKEN= # Cloudflare API token for DNS challenge (not used by default configuration)
# and set appropriate Traefik environment/command for DNS challenge (not configured in compose by default).

# Admin credentials are loaded from secrets/admin_credentials.env for qBittorrent and Grafana.
# Those credentials are defined in that file. If you prefer, you can override them here or edit that file.

# Grafana default admin (set in secrets file by default)
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=OTaEe2NeFpMw26Ju

# qBittorrent default WebUI login (set in secrets file by default)
WEBUI_USER=admin
WEBUI_PASS=ZPBgS1SyiW1Jvdv6

# If you change passwords above, update both here and in secrets/admin_credentials.env or vice versa.

Notes on .env: We included the key variables and left placeholders for you to fill (like DOMAIN, LETSENCRYPT_EMAIL). The randomly generated passwords shown here should be changed. The variables for Grafana and qBittorrent credentials appear both here (commented) and are loaded via the secrets file. For simplicity, you can edit the secrets/admin_credentials.env file directly to change those passwords.
Default Admin Credentials (secrets/admin_credentials.env)
This file is in the secrets/ directory and contains sensitive credentials. By using a separate file, we avoid putting plain-text passwords directly in the compose files. Here is the content with placeholders and defaults:
# Default admin credentials for various services.
# CHANGE these values immediately for security.

# Grafana Admin Account
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=OTaEe2NeFpMw26Ju

# qBittorrent Web UI Account
WEBUI_USER=admin
WEBUI_PASS=ZPBgS1SyiW1Jvdv6

# Note: Jellyfin and Jellyseerr accounts are created via their setup wizards (not preset here).
# Sonarr/Radarr/etc have no default login (they trust network access).
This file is referenced by the compose definitions for Grafana and qBittorrent. It allows those services to pick up these values as env vars. It’s good practice to store this file in a secure location (by default it’s in the repository, but if you publish your repo, be sure to remove or gitignore the secrets/ folder to avoid exposing your credentials). In a production deployment, you might use Docker secrets, but for simplicity, we’re using an env file which is easier to manage in Compose. Again, do not use the provided default passwords in a production environment. They are randomly generated for this template but since they are now known (public), you should treat them as examples. Choose your own strong passwords.
WireGuard VPN Configuration (vpn/wg0.conf)
The vpn/wg0.conf file should contain your WireGuard client configuration. We have included a placeholder for reference:
# WireGuard Client Configuration - Placeholder
# Replace the values below with those provided by your VPN service or your own server.

[Interface]
PrivateKey = YOUR_PRIVATE_KEY_GOES_HERE
Address = 10.13.37.2/24    # Example address, will be given by your VPN provider
DNS = 1.1.1.1

[Peer]
PublicKey = VPN_SERVER_PUBLIC_KEY_GOES_HERE
Endpoint = vpn-endpoint.example.com:51820   # e.g., server host and WireGuard port
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
Instructions: Remove the example lines and insert the actual config. Ensure no trailing spaces or invalid characters. Typically, your VPN provider will give you a .conf file that looks similar. You can paste it entirely here. This file is mounted into the Gluetun container, which will automatically establish the tunnel on startup. We’ve included DNS = 1.1.1.1 in the example; your provider might include their own DNS or you can choose one (1.1.1.1 is Cloudflare’s DNS). Keep this file secure. It contains your VPN credentials (in the form of keys). If someone obtains this, they could potentially use your VPN account.
License
This project is released under the MIT License. See the LICENSE file below for the full text.
MIT License

Copyright (c) 2025 the-ultimate-media-docker contributors

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights 
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell 
copies of the Software, and to permit persons to whom the Software is 
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in 
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR 
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES of MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE and NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES or OTHER 
LIABILITY, WHETHER IN AN ACTION of CONTRACT, TORT or OTHERWISE, ARISING 
FROM, OUT OF or IN CONNECTION with the SOFTWARE or the USE or OTHER DEALINGS 
IN THE SOFTWARE.
All Sources
