# Docker Homelab Setup Guide

Complete installation and configuration guide for running a full media server homelab using Docker Compose.

## Overview

This setup includes:
- **Jellyfin**: Media server for streaming movies/TV shows
- **Jellyseerr**: Request management system
- **Radarr**: Movie management and monitoring
- **Sonarr**: TV show management and monitoring  
- **Bazarr**: Subtitle management
- **qBittorrent**: Torrent client (via VPN)
- **Prowlarr**: Indexer manager
- **FlareSolverr**: Cloudflare bypass proxy
- **Gluetun**: VPN client with kill-switch
- **Homepage**: Dashboard for all services
- **Autoheal**: Container health monitoring

## Prerequisites

- Linux server with Docker support (Ubuntu/Debian recommended)
- VPN subscription (Surfshark, NordVPN, ExpressVPN, etc.)
- At least 4GB RAM and 20GB free disk space
- Basic command line knowledge

## Step 1: Install Docker

### Ubuntu/Debian Installation

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group
sudo usermod -aG docker $USER

# Logout and login again, or run:
newgrp docker

# Test Docker installation
docker --version
docker compose version
```

## Step 2: Setup Directory Structure

```bash
# Create main directory
mkdir -p ~/homelab
cd ~/homelab

# Create data directories
mkdir -p data/{downloads/complete,downloads/incomplete,media/{movies,tv,music}}

# Set proper permissions
sudo chown -R $USER:$USER data/
chmod -R 775 data/

# Verify structure
tree data/ || ls -la data/
```

Expected structure:
```
data/
├── downloads/
│   ├── complete/
│   └── incomplete/
└── media/
    ├── movies/
    ├── tv/
    └── music/
```

## Step 3: Get VPN Credentials

### For Surfshark (WireGuard)

1. Visit [Surfshark Account](https://my.surfshark.com)
2. Navigate to **VPN → Manual setup → WireGuard**
3. Choose your preferred location (e.g., Finland/Helsinki)
4. Generate configuration and note:
   - **Private Key** (long string)
   - **Address** (e.g., 10.14.0.2/16)

### For NordVPN (WireGuard)

1. Visit [NordVPN Account](https://my.nordaccount.com)
2. Go to **NordVPN → Manual setup → WireGuard**
3. Generate new keys and download config
4. Note the **PrivateKey** and **Address** values

### For Other VPN Providers

Check [Gluetun VPN Provider List](https://github.com/qdm12/gluetun/wiki) for configuration details.

## Step 4: Create Environment File

Create `.env` file in the homelab directory:

```bash
nano .env
```

Add the following content (customize the values):

```env
# Homelab Docker Environment Configuration

# General Configuration
TZ=Europe/Helsinki
PUID=1000
PGID=1000

# Data Storage Root Path
# Use full path to your data directory
DATA_ROOT=/home/yourusername/homelab/data

# VPN Configuration (Surfshark Example)
VPN_SERVICE_PROVIDER=surfshark
VPN_TYPE=wireguard

# Surfshark WireGuard Configuration
# Get these from your Surfshark account
WIREGUARD_PRIVATE_KEY=your_private_key_here
WIREGUARD_ADDRESSES=10.14.0.2/16

# Server Location
SERVER_COUNTRIES=Finland
SERVER_CITIES=Helsinki

# Network Configuration
FIREWALL_OUTBOUND_SUBNETS=172.20.0.0/24,192.168.0.0/16,10.0.0.0/8,172.16.0.0/12

# VPN Health Check
HEALTH_VPN_DURATION_INITIAL=120s

# DNS Configuration  
DNS_ADDRESS=162.252.172.57
```

**Important**: 
- Replace `your_private_key_here` with your actual WireGuard private key
- Update `DATA_ROOT` with the full path to your data directory
- Adjust timezone (`TZ`) to your location
- For different VPN providers, update the configuration accordingly

## Step 5: Download Docker Compose File

```bash
# Download the compose file
curl -o compose-full.yaml https://raw.githubusercontent.com/aarooh/homelab-setup/main/compose-full.yaml

# Or copy the compose-full.yaml file to your homelab directory
```

## Step 6: Start the Services

```bash
# Start all services
docker compose -f compose-full.yaml up -d

# Check if services are starting
docker compose -f compose-full.yaml ps

# Monitor logs (optional)
docker compose -f compose-full.yaml logs -f gluetun
```

## Step 7: Verify VPN Connection

```bash
# Test that VPN is working
docker exec gluetun wget -qO- https://ipinfo.io

# Should show your VPN server's IP, not your real IP
```

## Step 8: Service Configuration

### Access URLs

Once all containers are running, access the services at:

| Service | URL | Purpose |
|---------|-----|---------|
| **Homepage** | http://localhost:3000 | Dashboard |
| **Jellyfin** | http://localhost:8096 | Media server |
| **Jellyseerr** | http://localhost:5055 | Request system |
| **qBittorrent** | http://localhost:8080 | Torrent client |
| **Prowlarr** | http://localhost:9696 | Indexer manager |
| **FlareSolverr** | http://localhost:8191 | Cloudflare bypass |
| **Radarr** | http://localhost:7878 | Movie management |
| **Sonarr** | http://localhost:8989 | TV management |
| **Bazarr** | http://localhost:6767 | Subtitle management |

### qBittorrent Setup

1. **Get the admin password:**
   ```bash
   docker logs qbittorrent | grep -i password
   ```
   Look for: `The WebUI administrator password was not set. A temporary password is provided for this session:`

2. **Login and configure:**
   - URL: http://localhost:8080
   - Username: `admin`
   - Password: Use the temporary password from logs

3. **Essential settings:**
   - **Options → Downloads:**
     - Default Save Path: `/downloads/complete`
     - Keep incomplete torrents in: `/downloads/incomplete`
   - **Options → Advanced:**
     - Network Interface: `tun0` (forces VPN-only traffic)
   - **Options → Web UI:**
     - Change the default password

### Prowlarr Setup

1. Access http://localhost:9696
2. **Add Indexers:**
   - Go to **Indexers → Add Indexer**
   - Add popular indexers like 1337x, RARBG alternatives, etc.
   - For Cloudflare-protected sites, configure FlareSolverr:
     - **Settings → Indexers → FlareSolverr URL:** `http://172.20.0.11:8191`

3. **Configure Apps:**
   - Go to **Settings → Apps → Add Application**
   - Add Radarr: `http://172.20.0.12:7878`
   - Add Sonarr: `http://172.20.0.13:8989`
   - Get API keys from each service's **Settings → General**

### Radarr Setup

1. Access http://localhost:7878
2. **Add Download Client:**
   - **Settings → Download Clients → Add → qBittorrent**
   - Host: `172.20.0.10` (Gluetun container IP)
   - Port: `8080`
   - Username/Password: From qBittorrent setup

3. **Configure Paths:**
   - **Settings → Media Management → Root Folders → Add Root Folder**
   - Path: `/movies`

4. **Quality Profiles:**
   - **Settings → Profiles → Quality Profiles**
   - Adjust quality settings as needed

### Sonarr Setup

1. Access http://localhost:8989
2. **Add Download Client:** Same as Radarr
3. **Configure Paths:**
   - **Settings → Media Management → Root Folders → Add Root Folder**
   - Path: `/tv`

### Bazarr Setup

1. Access http://localhost:6767
2. **Connect to Sonarr/Radarr:**
   - **Settings → Sonarr → Add**
     - Address: `http://172.20.0.13`
     - Port: `8989`
     - API Key: From Sonarr settings
   - **Settings → Radarr → Add**
     - Address: `http://172.20.0.12`
     - Port: `7878`
     - API Key: From Radarr settings

3. **Configure Providers:**
   - **Settings → Providers → Add Subtitle Provider**
   - Add OpenSubtitles, Addic7ed, etc.

### Jellyfin Setup

1. Access http://localhost:8096
2. **Initial Setup Wizard:**
   - Create admin user
   - Add media libraries:
     - Movies: `/data/movies`
     - TV Shows: `/data/tvshows`
     - Music: `/data/music`
   - Configure metadata providers

3. **Hardware Acceleration (Optional):**
   - **Dashboard → Playback → Transcoding**
   - Enable hardware acceleration if supported

### Jellyseerr Setup

1. Access http://localhost:5055
2. **Connect to Jellyfin:**
   - Jellyfin URL: `http://172.20.0.15:8096`
   - Use your Jellyfin admin credentials

3. **Connect to Radarr/Sonarr:**
   - **Settings → Services**
   - Add Radarr: `http://172.20.0.12:7878`
   - Add Sonarr: `http://172.20.0.13:8989`
   - Use API keys from respective services

### Homepage Dashboard Setup

1. Access http://localhost:3000
2. Configure widgets and bookmarks
3. Edit the configuration files if needed:
   ```bash
   docker exec -it homepage sh
   ```

## Step 9: Test the Complete Workflow

1. **Request Content:**
   - Open Jellyseerr (http://localhost:5055)
   - Search for a movie/TV show
   - Submit request

2. **Monitor Progress:**
   - Check Radarr/Sonarr for the added item
   - Verify qBittorrent starts downloading
   - Confirm files appear in media directories

3. **Verify in Jellyfin:**
   - Content should appear in Jellyfin after download completes
   - Test playback

## Troubleshooting

### VPN Not Connecting
```bash
# Check Gluetun logs
docker logs gluetun

# Test VPN connectivity
docker exec gluetun ping 8.8.8.8
```

### qBittorrent Can't Access Internet
```bash
# Verify network binding
docker exec qbittorrent ip route

# Should show tun0 interface
```

### Permission Issues
```bash
# Fix data directory permissions
sudo chown -R 1000:1000 ~/homelab/data
chmod -R 775 ~/homelab/data
```

### Container Won't Start
```bash
# Check specific container logs
docker logs <container_name>

# Restart specific service
docker compose -f compose-full.yaml restart <service_name>
```

### High Memory Usage
```bash
# Monitor resource usage
docker stats

# Restart services if needed
docker compose -f compose-full.yaml restart
```

## Maintenance

### Regular Updates
```bash
# Update all containers
docker compose -f compose-full.yaml pull
docker compose -f compose-full.yaml up -d

# Clean up old images
docker image prune -a
```

### Backup Configuration
```bash
# Backup Docker volumes
docker run --rm -v homelab_jellyfin_config:/data -v $(pwd):/backup alpine tar czf /backup/jellyfin-backup.tar.gz -C /data .

# Backup compose file and environment
cp compose-full.yaml .env ~/backups/
```

### Monitor Logs
```bash
# Monitor all services
docker compose -f compose-full.yaml logs -f

# Monitor specific service
docker logs -f <container_name>
```
