## Home Server Setup Guide

---

### 1. Prerequisites

* Docker and Docker Compose installed
* Cloudflared CLI installed and authenticated
* Valid domains for each service (e.g., `jellyfin.example.com`)
* `.env` file configured (see [Environment Variables](#5-environment-variables))

---

### 2. Quick Start

```bash
# Launch all services in detached mode
docker compose --env-file .env up -d

# Verify that qBittorrent is using the VPN
docker exec -it qbittorrent curl ifconfig.io
```

---

### 3. Cloudflared Tunnel Setup

1. **Create a new tunnel**

   ```bash
   cloudflared tunnel create home-server
   ```

   * Note the generated credentials file (e.g., `123-123-123-123-123.json`).

2. **Prepare Cloudflared config**

   * Copy the credentials file into your config directory:

     ```text
     ./config/cloudflared/123-123-123-123-123.json
     ```
   * Create `config.yml` alongside it:

     ```yaml
     tunnel: 123-123-123-123-123
     credentials-file: /etc/cloudflared/123-123-123-123-123.json

     ingress:
       - hostname: jellyfin.example.com
         service: http://jellyfin:8096

       - hostname: sonarr.example.com
         service: http://sonarr:8989

       - hostname: radarr.example.com
         service: http://radarr:7878

       - hostname: lidarr.example.com
         service: http://lidarr:8686

       - hostname: prowlarr.example.com
         service: http://prowlarr:9696

       - hostname: jellyseerr.example.com
         service: http://jellyseerr:5055

       - hostname: qbittorrent.example.com
         service: http://gluetun:8880

       - hostname: seafile.example.com
         service: http://seafile:80

       - service: http_status:404
     ```

3. **Configure DNS routes**

   ```bash
   cloudflared tunnel route dns home-server jellyfin.example.com
   cloudflared tunnel route dns home-server emby.example.com
   cloudflared tunnel route dns home-server sonarr.example.com
   cloudflared tunnel route dns home-server radarr.example.com
   cloudflared tunnel route dns home-server lidarr.example.com
   cloudflared tunnel route dns home-server prowlarr.example.com
   cloudflared tunnel route dns home-server jellyseerr.example.com
   cloudflared tunnel route dns home-server qbittorrent.example.com
   cloudflared tunnel route dns home-server seafile.example.com
   ```

---

### 4. Seafile Reverse-Proxy Configuration

When fronting Seafile with Cloudflared (or any reverse proxy), update the `CSRF_TRUSTED_ORIGINS` setting:

1. **Extract `seahub_settings.py`**

   ```bash
   docker cp seafile:/shared/seafile/conf/seahub_settings.py seahub_settings.py
   ```

2. **Edit `seahub_settings.py`**

   ```python
   CSRF_TRUSTED_ORIGINS = [
       "https://seafile.example.com",
   ]
   ```

3. **Reapply to container**

   ```bash
   docker cp seahub_settings.py seafile:/shared/seafile/conf/seahub_settings.py
   ```

4. **Extract `seafdav.conf`**

  ```bash
  docker cp seafile:/opt/seafile/conf/seafdav.conf seafdav.conf
  ```

5. **Edit `seafdav.conf`**

   ```python
   enabled = true
   ```

6. **Reapply to container**

   ```bash
   docker cp seafdav.conf seafile:/opt/seafile/conf/seafdav.conf
   ```

---

### 5. Docker Compose Configuration Overview

Below is a high-level overview of the `docker-compose.yaml` setup, illustrating key services and networks.

```yaml
services:
  # Media management
  sonarr:
    image: linuxserver/sonarr
    volumes:
      - ${CONFIG_DIR}/sonarr:/config
      - ${SHOWS_DIR}:/shows
      - ${DOWNLOADS_DIR}:/downloads
  radarr:
    image: linuxserver/radarr
    # ... similar structure for lidarr, prowlarr, jellyseerr

  # VPN client
  gluetun:
    image: qmcgaw/gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun

  # Torrent client behind VPN
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent
    network_mode: "service:gluetun"

  # Reverse proxy tunnel
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run
    volumes:
      - ./cloudflared:/etc/cloudflared:ro

  # Media server
  jellyfin:
    image: jellyfin/jellyfin
    ports:
      - "8096:8096"
      - "8920:8920"

  # Seafile stack
  mariadb:
    image: mariadb:10.11
  memcached:
    image: memcached:1.6-alpine
  seafile:
    image: seafileltd/seafile-mc:11.0-latest
    environment:
      - DB_HOST=mariadb
      - CSRF_TRUSTED_ORIGINS=["https://seafile.${SERVER_HOSTNAME}"]
    tmpfs:
      - /opt/seafile/pids

networks:
  default:
    driver: bridge
  seafile:
    driver: bridge

volumes:
  seafile-data:
  mariadb-data:
```

---

### 6. Environment Variables

Create an `.env` file (or copy from `.env.example`) with the following entries:

```dotenv
# User and Paths
PUID=1000
PGID=1000
TZ=Asia/Jerusalem
CONFIG_DIR=./config
SHOWS_DIR=/path/to/media/shows
MOVIES_DIR=/path/to/media/movies
MUSIC_DIR=/path/to/media/music
DOWNLOADS_DIR=/path/to/media/downloads
SQL_DIR=/path/to/sql
SEAFILE_DIR=/path/to/seafile

# VPN Credentials
VPN_COUNTRY=Israel
NORD_USER=<your_nordvpn_username>
NORD_PASS=<your_nordvpn_password>

# Hostname and Seafile Admin
SERVER_HOSTNAME=example.com
SEAFILE_ADMIN_EMAIL=admin@example.com
SEAFILE_ADMIN_PASSWORD=<strong_password>
SEAFILE_SERVER_LETSENCRYPT=false

# Database
MYSQL_ROOT_PASSWORD=<your_mysql_root_password>
```

### 7. Setup and configure your applications
[![Setup and configure your applications](https://img.youtube.com/vi/1eDUkmwDrWU/0.jpg)](https://youtu.be/1eDUkmwDrWU)
