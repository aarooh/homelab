# Homelab Setup

**Helper Scripts**: [ProxmoxVE Community Scripts](https://community-scripts.github.io/ProxmoxVE/scripts?id=update-lxcs)

## 1. Proxmox

### Post Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/pve/post-pve-install.sh)"
```

### My Proxmox Setup Summary

#### Initial Configuration

**Post-Install Setup:**
- Disabled enterprise repositories and enabled no-subscription repos
- Deleted local-lvm and resized local storage to use full boot drive space
- Enabled IOMMU for hardware passthrough capabilities

#### Storage Architecture

Created two ZFS pools for optimal performance:

- **Flash Pool**: NVME SSDs in mirror configuration for VMs/containers
- **Tank Pool**: HDDs in RAIDZ for bulk media storage

#### Container Setup (Media Server - CT 100)

**Base Configuration:**
- **OS**: Ubuntu 22.04 LXC container
- **DHCP IP**: 10.0.0.100/24 (matching CT ID for organization)
- **Resources**: 6 CPU cores, 8GB RAM, 64GB system storage

**Mount Points Added:**
- **Data Mount**: `/data` - 4TB from tank pool (bulk media storage)
- **Docker Mount**: `/docker` - 128GB from flash pool (container data)

#### SMB Share Setup

Configured Samba for network access:
- Created dedicated user with proper permissions
- Set up shares for both `/data` and `/docker` directories
- Enabled Windows discovery with wsdd
- Configured firewall rules for SMB access

#### Network Configuration

- Used DHCP for IP's
- Configured proper subnet access (10.0.0.0/24)
- Ensured containers can communicate with each other

#### Key Benefits of This Setup

- **Performance**: Direct ZFS storage access, no network overhead
- **Organization**: CT IDs match IP addresses for easy identification
- **Flexibility**: Separate pools for different workload types
- **Accessibility**: SMB shares allow easy file management from other devices
- **Scalability**: Can easily add more containers with dedicated mount points

**Guide**: [Storage README](https://github.com/TechHutTV/homelab/blob/main/storage/README.md)

## 2. Jellyfin

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/jellyfin.sh)"
```

### Configuration

- **FFmpeg path**: `/usr/lib/jellyfin-ffmpeg/ffmpeg`
- **Config file location**: `/etc/jellyfin/`
- **Update source**: [Jellyfin Script](https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/jellyfin.sh)

## 3. Homepage

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/homepage.sh)"
```

### Additional Code Editor

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/tools/addon/coder-code-server.sh)"
```

## 4. Jellyseer

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/jellyseerr.sh)"
```

## 5. Media Downloader (Torrent + VPN) - CT 104

### Overview

Secure torrenting setup with VPN kill-switch using Gluetun, qBittorrent, Prowlarr, and FlareSolverr in a privileged LXC container.

### Network Isolation Setup

⚠️ **Important**: Due to Gluetun crashes causing system-wide network failures, this container is configured on an isolated network bridge (vmbr1) to prevent taking down the entire Proxmox network.

**Network Configuration Guide**: See [networking.md](networking.md) for detailed instructions on:
- Creating isolated bridge (vmbr1) for media-dl container
- Configuring port forwarding for external access
- Network isolation benefits and troubleshooting

### Prerequisites

- Active Surfshark VPN subscription
- ZFS storage pool for media data
- Basic understanding of Docker and LXC
- **Recommended**: Complete network isolation setup from [networking.md](networking.md)

### Step 1: Create Privileged LXC Container

In Proxmox Web Interface:

**Create Container:**
- **CT ID**: 104
- **Hostname**: media-dl
- **Template**: Ubuntu 22.04
- **Root Disk**: 64GB (local storage)
- **CPU**: 4 cores
- **Memory**: 4GB RAM, 1GB swap
- **Network**: DHCP on vmbr0
- **⚠️ IMPORTANT**: Uncheck "Unprivileged container"

Don't start yet - configure mount points first

### Step 2: Configure LXC Mount Points and VPN Support

```bash
# Stop container (if running)
pct stop 104

# Edit LXC configuration
nano /etc/pve/lxc/104.conf
```

**Complete configuration:**

```ini
arch: amd64
cores: 4
hostname: media-dl
memory: 4096
mp0: disks:subvol-100-disk-0,mp=/data,backup=1,size=3800G
net0: name=eth0,bridge=vmbr0,ip=dhcp,type=veth
ostype: ubuntu
rootfs: local:104/vm-104-disk-0.raw,size=64G
swap: 1024
tags: media
unprivileged: 0
lxc.cgroup2.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file
lxc.apparmor.profile = unconfined
```

**Key additions:**
- `unprivileged: 0` - Makes container privileged
- VPN device access lines for TUN/TAP support
- AppArmor unconfined for Docker compatibility
- Shared data mount from existing media storage

### Step 3: Start Container and Install Docker

```bash
# Start container
pct start 104

# Enter container
pct enter 104

# Update system
apt update && apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
apt install docker-compose-plugin -y
systemctl enable docker
systemctl start docker
```

### Step 4: Setup Application Structure

```bash
# Create directory in root (simpler approach)
cd /root

# Download official TechHutTV compose files
wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/compose.yaml
wget https://github.com/TechHutTV/homelab/raw/refs/heads/main/media/.env

# Set proper permissions
chown -R 1000:1000 /root
```

### Step 5: Get Surfshark VPN Credentials

1. Login to Surfshark account: [https://my.surfshark.com](https://my.surfshark.com)
2. Navigate to: **VPN → Manual setup → WireGuard**
3. Generate configuration for preferred location (e.g., Finland/Helsinki)
4. Note down:
   - **PrivateKey** (long string)
   - **Address** (e.g., 10.14.0.2/16)

### Step 6: Configure Environment File

```bash
nano .env
```

Replace entire content with:

```env
# General UID/GID and Timezone
TZ=Europe/Helsinki
PUID=1000
PGID=1000

# Surfshark VPN Configuration
VPN_SERVICE_PROVIDER=surfshark
VPN_TYPE=wireguard

# From your Surfshark WireGuard config
WIREGUARD_PRIVATE_KEY=your_private_key_here
WIREGUARD_ADDRESSES=10.14.0.2/16

# Server location
SERVER_COUNTRIES=Finland
SERVER_CITIES=Helsinki

# Health check duration
HEALTH_VPN_DURATION_INITIAL=120s

# Security settings
FIREWALL_OUTBOUND_SUBNETS=172.39.0.0/24,192.168.1.0/24,10.0.0.0/8,172.16.0.0/12
DNS_ADDRESS=162.252.172.57
DOT=off
IPV6=off
```

```bash
chmod 600 .env
```

### Step 7: Configure Docker Compose

```bash
nano compose.yaml
```

Remove unnecessary services and add AppArmor fixes:

```yaml
networks:
  servarrnetwork:
    name: servarrnetwork
    ipam:
      config:
        - subnet: 172.39.0.0/24

services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    security_opt:
      - apparmor:unconfined
    privileged: true
    networks:
      servarrnetwork:
        ipv4_address: 172.39.0.2
    ports:
      - 8080:8080 # qbittorrent
      - 6881:6881 # qbittorrent torrent port
      - 9696:9696 # prowlarr
    volumes:
      - ./gluetun:/gluetun
    env_file:
      - .env
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 20s
      timeout: 10s
      retries: 5
    restart: unless-stopped

  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined
    privileged: true
    labels:
      - deunhealth.restart.on.unhealthy=true
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ./qbittorrent:/config
      - /data:/data
    depends_on:
      gluetun:
        condition: service_healthy
        restart: true
    network_mode: service:gluetun
    healthcheck:
      test: ping -c 1 www.google.com || exit 1
      interval: 60s
      retries: 3
      start_period: 20s
      timeout: 10s

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    security_opt:
      - apparmor:unconfined
    privileged: true
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./prowlarr:/config
    restart: unless-stopped
    depends_on:
      gluetun:
        condition: service_healthy
        restart: true
    network_mode: service:gluetun

  deunhealth:
    image: qmcgaw/deunhealth
    container_name: deunhealth
    network_mode: "none"
    security_opt:
      - apparmor:unconfined
    privileged: true
    environment:
      - LOG_LEVEL=info
      - HEALTH_SERVER_ADDRESS=127.0.0.1:9999
      - TZ=${TZ}
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  flaresolverr:
    image: ghcr.io/flaresolverr/flaresolverr:latest
    container_name: flaresolverr
    security_opt:
      - apparmor:unconfined
    privileged: true
    environment:
      - LOG_LEVEL=info
      - LOG_HTML=false
      - CAPTCHA_SOLVER=none
      - TZ=${TZ}
    ports:
      - 8191:8191
    restart: unless-stopped
    networks:
      servarrnetwork:
        ipv4_address: 172.39.0.3
```

### Step 8: Start and Test

```bash
# Start all services
docker compose up -d

# Check container status
docker compose ps

# Test VPN connection (should show VPN IP, not real IP)
docker exec gluetun wget -qO- https://ipinfo.io

# Test qBittorrent VPN (should match Gluetun IP)
docker exec qbittorrent wget -qO- https://ipinfo.io/ip
```

### Step 9: Configure Applications

**Access URLs** (replace with your container's IP):
- **qBittorrent**: http://[container-ip]:8080
- **Prowlarr**: http://[container-ip]:9696
- **FlareSolverr**: http://[container-ip]:8191

**Get qBittorrent password:**

```bash
docker logs qbittorrent | grep -i password
```

#### qBittorrent Configuration

1. Login with `admin` and generated password
2. **Tools → Options → Downloads:**
   - Default Save Path: `/data/downloads/qbittorrent/completed`
   - Keep incomplete: `/data/downloads/qbittorrent/incomplete`
3. **Tools → Options → Advanced → Network Interface**: Set to `tun0`
4. Change default password in WebUI settings

#### Prowlarr Configuration

1. Add indexers for your region
2. Configure API keys for future Sonarr/Radarr integration
3. For indexers with Cloudflare protection, configure FlareSolverr:
   - **FlareSolverr URL**: http://172.39.0.3:8191
   - **Tags**: Add "flaresolverr" tag to indexers that need it

#### FlareSolverr Configuration

FlareSolverr is a proxy server that bypasses Cloudflare protection for indexers that require it. It runs automatically and doesn't require manual configuration. Simply reference it in Prowlarr when setting up indexers that are protected by Cloudflare.

### Security Features

- ✅ **VPN Kill-Switch**: If VPN fails, containers lose internet access (no IP leaks)
- ✅ **Network Isolation**: qBittorrent can only access internet through VPN
- ✅ **Automatic Restart**: deunhealth monitors and restarts failed containers
- ✅ **Interface Binding**: qBittorrent locked to VPN interface only
- ✅ **Health Monitoring**: Comprehensive VPN connectivity checks

### Troubleshooting

**Test Kill-Switch:**

```bash
# Stop VPN
docker stop gluetun

# Try internet access (should fail)
docker exec qbittorrent wget --timeout=10 -qO- https://ipinfo.io/ip

# Restart VPN
docker start gluetun
```

**Common Issues:**
- **AppArmor errors**: Ensure `security_opt` and `privileged: true` in all services
- **VPN connection fails**: Verify Surfshark credentials in `.env` file
- **No internet**: Check TUN device with `ls -la /dev/net/tun`

## 6. Radarr

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/radarr.sh)"
```

### Configuration

- **CT ID**: 105
- **Access**: http://[container-ip]:7878
- **Purpose**: Movie collection management

### Mount Point Configuration

After installation, add the shared storage mount point:

```bash
# Stop the container
pct stop 105

# Edit LXC configuration
nano /etc/pve/lxc/105.conf

# Add this line to the configuration:
mp0: disks:subvol-100-disk-0,mp=/data,backup=1,size=3800G

# Start the container
pct start 105
```

### Setup Steps

1. **Add Download Client**: Point to qBittorrent on media-dl container
   - **Host**: [media-dl-ip] (e.g., 192.168.1.109)
   - **Port**: 8080
   - **Username/Password**: From qBittorrent setup

2. **Add Indexers**: Connect to Prowlarr for automatic sync
   - **Prowlarr URL**: http://[media-dl-ip]:9696
   - **API Key**: Get from Prowlarr settings

3. **Configure Paths:**
   - **Movies**: `/data/movies`
   - **Downloads**: `/data/downloads/qbittorrent/completed`

## 7. Sonarr

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/sonarr.sh)"
```

### Configuration

- **CT ID**: 106
- **Access**: http://[container-ip]:8989
- **Purpose**: TV show collection management

### Mount Point Configuration

After installation, add the shared storage mount point:

```bash
# Stop the container
pct stop 106

# Edit LXC configuration
nano /etc/pve/lxc/106.conf

# Add this line to the configuration:
mp0: disks:subvol-100-disk-0,mp=/data,backup=1,size=3800G

# Start the container
pct start 106
```

### Setup Steps

1. **Add Download Client**: Same as Radarr setup
2. **Add Indexers**: Connect to Prowlarr
3. **Configure Paths:**
   - **TV Shows**: `/data/shows`
   - **Downloads**: `/data/downloads/qbittorrent/completed`

## 8. Bazarr (Optional)

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/bazarr.sh)"
```

### Configuration

- **CT ID**: 107
- **Access**: http://[container-ip]:6767
- **Purpose**: Subtitle management for Sonarr/Radarr

## 9. Huntarr

### Install Script

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/configarr.sh)"
```

### Configuration

- **CT ID**: 108
- **Access**: http://[container-ip]:9705
- **Purpose**: Configuration management and automation for *arr services

### Mount Point Configuration

After installation, add the shared storage mount point:

```bash
# Stop the container
pct stop 108

# Edit LXC configuration
nano /etc/pve/lxc/108.conf

# Add this line to the configuration:
mp0: disks:subvol-100-disk-0,mp=/data,backup=1,size=3800G

# Start the container
pct start 108
```

## 10. Additional Services (Optional)

### Lidarr (Music Management)

**Install Script:**

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/lidarr.sh)"
```

- **CT ID**: 109
- **Access**: http://[container-ip]:8686

### Readarr (Book Management)

**Install Script:**

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/readarr.sh)"
```

- **CT ID**: 110
- **Access**: http://[container-ip]:8787

## Backup Configuration

### Fix: Use the Correct ZFS Path

```bash
# Remove the wrong backup folder
rm -rf /data

# Create backup script in the correct location (ZFS disk)
mkdir -p /disks/subvol-100-disk-0/backup

# Create the backup script in the right place
nano /disks/subvol-100-disk-0/backup/backup-configs.sh
```

### Updated Script with Correct Paths

```bash
#!/bin/bash
# Simple Homelab Configuration Backup

BACKUP_DIR="/disks/subvol-100-disk-0/backup"
DATE=$(date +%Y-%m-%d_%H-%M)

echo "Starting backup at $(date)"

# Create timestamped backup directory
mkdir -p "$BACKUP_DIR/$DATE"

# Backup LXC configurations
echo "Backing up LXC configurations..."
mkdir -p "$BACKUP_DIR/$DATE/lxc-configs"
cp /etc/pve/lxc/*.conf "$BACKUP_DIR/$DATE/lxc-configs/"

# Backup Community Scripts configurations  
echo "Backing up Community Scripts..."
mkdir -p "$BACKUP_DIR/$DATE/community-scripts"
cp /opt/community-scripts/*.conf "$BACKUP_DIR/$DATE/community-scripts/"

# Backup Docker configs from media-dl
echo "Backing up Docker configurations..."
mkdir -p "$BACKUP_DIR/$DATE/docker-configs"
pct exec 104 -- cp /root/compose.yaml /data/backup/$DATE/docker-configs/ 2>/dev/null
pct exec 104 -- cp /root/.env /data/backup/$DATE/docker-configs/env-backup 2>/dev/null

# Create summary
cat > "$BACKUP_DIR/$DATE/summary.txt" << EOF
Homelab Backup - $DATE

LXC Containers:
$(pct list)

Community Scripts:
$(ls -1 /opt/community-scripts/*.conf | xargs -I {} basename {} .conf)

Backup Contents:
- lxc-configs/: All LXC container configurations
- community-scripts/: All community script configurations  
- docker-configs/: Docker compose and environment files
EOF

echo "Backup completed: $BACKUP_DIR/$DATE"
echo "Total size: $(du -sh $BACKUP_DIR/$DATE | cut -f1)"
```

### Make Executable and Run

```bash
chmod +x /disks/subvol-100-disk-0/backup/backup-configs.sh

# Run the backup
/disks/subvol-100-disk-0/backup/backup-configs.sh
```

### Your Backup Structure

```
/disks/subvol-100-disk-0/backup/
├── backup-configs.sh
├── 2024-07-05_15-30/
│   ├── summary.txt
│   ├── lxc-configs/
│   │   ├── 100.conf
│   │   ├── 101.conf
│   │   ├── 102.conf
│   │   ├── 103.conf
│   │   ├── 104.conf
│   │   └── 105.conf
│   ├── community-scripts/
│   │   ├── homarr.conf
│   │   ├── homepage.conf
│   │   ├── jellyfin.conf
│   │   ├── jellyseerr.conf
│   │   ├── radarr.conf
│   │   ├── recyclarr.conf
│   │   ├── sonarr.conf
│   │   └── wizarr.conf
│   └── docker-configs/
│       ├── compose.yaml
│       └── env-backup
└── 2024-07-06_10-15/
    └── (next backup...)
```

### Optional: Automatic Monthly Backup

```bash
# Add to crontab for automatic backups
crontab -e

# Add this line for monthly backups at 1 AM on the 1st of each month
0 1 1 * * /disks/subvol-100-disk-0/backup/backup-configs.sh
```

### Access Your Backups

You can access backups from:

- **Proxmox host**: Direct access at `/disks/subvol-100-disk-0/backup/`
- **Any LXC with `/data` mount**: Access via `/data/backup/`
- **SMB share** (if configured): Navigate to backup folder

Simple, clean, and accessible from anywhere in your homelab!

## Complete Media Stack Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Media-dl      │    │     Sonarr      │    │     Radarr      │
│   (CT 104)      │    │    (CT 106)     │    │    (CT 105)     │
│                 │    │                 │    │                 │
│ • Gluetun+VPN   │◄───┤ • TV Shows      │    │ • Movies        │
│ • qBittorrent   │    │ • Monitoring    │    │ • Monitoring    │
│ • Prowlarr      │    │ • Requests      │    │ • Requests      │
│ • FlareSolverr  │    │                 │    │                 │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │   Shared ZFS    │
                    │     Storage     │
                    │                 │
                    │ • /data/movies  │
                    │ • /data/shows   │
                    │ • /data/music   │
                    │ • /data/downloads│
                    └─────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         │                       │                       │
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    Jellyfin     │    │    Homepage     │    │   Jellyseerr    │
│    (CT 101)     │    │    (CT 102)     │    │    (CT 103)     │
│                 │    │                 │    │                 │
│ • Media Server  │    │ • Dashboard     │    │ • Requests      │
│ • Streaming     │    │ • Monitoring    │    │ • User Portal   │
│ • Transcoding   │    │ • Links         │    │ • Integration   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

## Integration Workflow

1. **User Requests** → Jellyseerr → Sonarr/Radarr
2. **Content Search** → Prowlarr finds torrents
3. **Secure Download** → qBittorrent via VPN
4. **File Management** → Sonarr/Radarr organize files
5. **Media Serving** → Jellyfin streams to devices
6. **Monitoring** → Homepage dashboard overview

## Key Benefits

- ✅ **Security**: All downloads via VPN with kill-switch
- ✅ **Automation**: Fully automated media acquisition
- ✅ **Organization**: Clean file structure and naming
- ✅ **Access**: Web interfaces for all services
- ✅ **Scalability**: Easy to add more services
- ✅ **Performance**: Dedicated containers for each service
- ✅ **Reliability**: Automatic restart and health monitoring

## Quick Access URLs Summary

| Service | Purpose | URL |
|---------|---------|-----|
| **Media-dl** | Downloads | http://[ip]:8080 (qBittorrent)<br>http://[ip]:9696 (Prowlarr)<br>http://[ip]:8191 (FlareSolverr) |
| **Jellyfin** | Streaming | http://[ip]:8096 |
| **Homepage** | Dashboard | http://[ip]:3000 |
| **Jellyseerr** | Requests | http://[ip]:5055 |
| **Radarr** | Movies | http://[ip]:7878 |
| **Sonarr** | TV Shows | http://[ip]:8989 |
| **Bazarr** | Subtitles | http://[ip]:6767 |
| **Huntarr** | Configuration | http://[ip]:9705 |

---

