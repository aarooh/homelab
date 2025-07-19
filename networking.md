# Proxmox Isolated Network Setup for Media-dl Container

## Overview

This tutorial shows how to create an isolated network bridge in Proxmox to prevent Docker container crashes (specifically Gluetun VPN) from taking down your entire Proxmox network. This setup provides network isolation while maintaining external access through port forwarding.

## Problem Statement

**Original Issue**: Gluetun container crashes caused complete Proxmox system lockups requiring manual restarts.

**Root Cause**: 
- Gluetun crash → Docker network cleanup failures
- Bridge table corruption → Broadcast storms 
- System-wide network failure

## Solution Architecture

```
Internet
    ↓
Router (192.168.1.1)
    ↓
Proxmox Host (192.168.1.110)
    ├── vmbr0 (Main Bridge) - Other containers/VMs
    └── vmbr1 (Isolated Bridge) - Media-dl container
            ↓
        CT104 (192.168.100.2) - Media downloading stack
```

## Prerequisites

- Proxmox VE host with existing containers
- Docker-compose setup with Gluetun + qBittorrent + Prowlarr
- Administrative access to Proxmox host

## Step 1: Create Isolated Bridge (vmbr1)

### 1.1 Configure Network Interface

Edit `/etc/network/interfaces` on the Proxmox host:

```bash
nano /etc/network/interfaces
```

Add the following configuration:

```bash
auto lo
iface lo inet loopback

auto vmbr0
iface vmbr0 inet dhcp
        bridge-ports eno2
        bridge-stp on
        bridge-fd 2
        bridge-hello 1
        bridge-maxage 6
        bridge-ageing 30

auto vmbr1
iface vmbr1 inet static
        address 192.168.100.1/24
        bridge-ports none
        bridge-stp on
        bridge-fd 15
        bridge-hello 2
        bridge-maxage 12
        post-up iptables -t nat -A POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 192.168.100.0/24 -o vmbr0 -j MASQUERADE

iface wlo1 inet manual
source /etc/network/interfaces.d/*
```

### 1.2 Key Configuration Details

- **bridge-ports none**: Creates virtual-only bridge (no physical interface)
- **Static IP**: 192.168.100.1/24 - Bridge acts as gateway for isolated network
- **STP enabled**: Spanning Tree Protocol prevents network loops
- **NAT rules**: Enable internet access for isolated network via masquerading

### 1.3 Enable IP Forwarding

```bash
echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf
sysctl -p
```

### 1.4 Apply Network Configuration

```bash
systemctl restart networking
# or reboot for safety
reboot
```

## Step 2: Move Container to Isolated Network

### 2.1 Change Container Network Settings

In Proxmox Web UI:
1. Navigate to **CT104** → **Hardware** → **Network Device (eth0)**
2. Change **Bridge** from `vmbr0` to `vmbr1`
3. Set **IPv4/CIDR** to `192.168.100.2/24`
4. Set **Gateway** to `192.168.100.1`

### 2.2 Restart Container

```bash
pct restart 104
```

### 2.3 Verify Network Configuration

```bash
# Enter container
pct enter 104

# Check IP configuration
ip addr show eth0
# Should show: 192.168.100.2/24

# Test connectivity
ping 192.168.100.1  # Bridge gateway
ping 8.8.8.8        # Internet via NAT

exit
```

## Step 3: Configure Port Forwarding

### 3.1 Add DNAT Rules for Service Access

```bash
# Forward qBittorrent (port 8080)
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.100.2:8080

# Forward Prowlarr (port 9696)
iptables -t nat -A PREROUTING -p tcp --dport 9696 -j DNAT --to-destination 192.168.100.2:9696

# Forward FlareSolverr (port 8191)
iptables -t nat -A PREROUTING -p tcp --dport 8191 -j DNAT --to-destination 192.168.100.2:8191

# Forward Code Server (port 8680)
iptables -t nat -A PREROUTING -p tcp --dport 8680 -j DNAT --to-destination 192.168.100.2:8680

# Forward Gluetun Web UI (port 8000)
iptables -t nat -A PREROUTING -p tcp --dport 8000 -j DNAT --to-destination 192.168.100.2:8000
```

### 3.2 Add FORWARD Rules

```bash
# Allow forwarded traffic to container
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 8080 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 9696 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 8191 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 8680 -j ACCEPT
iptables -A FORWARD -p tcp -d 192.168.100.2 --dport 8000 -j ACCEPT

# Allow return traffic
iptables -A FORWARD -p tcp -s 192.168.100.2 --sport 8080 -j ACCEPT
iptables -A FORWARD -p tcp -s 192.168.100.2 --sport 9696 -j ACCEPT
iptables -A FORWARD -p tcp -s 192.168.100.2 --sport 8191 -j ACCEPT
iptables -A FORWARD -p tcp -s 192.168.100.2 --sport 8680 -j ACCEPT
iptables -A FORWARD -p tcp -s 192.168.100.2 --sport 8000 -j ACCEPT
```

### 3.3 Alternative Broader Rules (if specific rules don't work)

```bash
iptables -I FORWARD -s 192.168.1.0/24 -d 192.168.100.0/24 -j ACCEPT
iptables -I FORWARD -s 192.168.100.0/24 -d 192.168.1.0/24 -j ACCEPT
```

## Step 4: Make Configuration Persistent

### 4.1 Save iptables Rules

```bash
# Install iptables-persistent
apt update && apt install iptables-persistent

# Save current rules
iptables-save > /etc/iptables/rules.v4
```

### 4.2 Verify Rules Persist After Reboot

```bash
# Check NAT rules
iptables -t nat -L POSTROUTING | grep 192.168.100
iptables -t nat -L PREROUTING | grep 192.168.100

# Check FORWARD rules
iptables -L FORWARD | grep 192.168.100
```

## Step 5: Verification and Testing

### 5.1 Test Direct Access from Proxmox Host

```bash
curl -I http://192.168.100.2:8080  # qBittorrent
curl -I http://192.168.100.2:9696  # Prowlarr
```

### 5.2 Test Port Forwarding

```bash
curl -I http://192.168.1.110:8080  # qBittorrent
curl -I http://192.168.1.110:9696  # Prowlarr
curl -I http://192.168.1.110:8191  # FlareSolverr
curl -I http://192.168.1.110:8680  # Code Server
curl -I http://192.168.1.110:8000  # Gluetun Web UI
```

### 5.3 Test External Access

From any device on the main network (192.168.1.x):
- **qBittorrent**: `http://192.168.1.110:8080`
- **Prowlarr**: `http://192.168.1.110:9696`
- **FlareSolverr**: `http://192.168.1.110:8191`
- **Code Server**: `http://192.168.1.110:8680`
- **Gluetun Web UI**: `http://192.168.1.110:8000`

## Docker Compose Configuration

Your existing Docker compose should work without changes. Here's the reference configuration:

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
      - 8080:8080
      - 6881:6881
      - 9696:9696
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
    image: lscr.io/linuxserver/qbittorrent:5.1.0
    container_name: qbittorrent
    restart: unless-stopped
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
    security_opt:
      - apparmor:unconfined
    privileged: true

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
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
    security_opt:
      - apparmor:unconfined
    privileged: true

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
```

## Benefits Achieved

### ✅ Network Isolation
- Gluetun crashes only affect isolated network (vmbr1)
- Main network (vmbr0) remains stable and accessible
- Other containers/VMs unaffected by media-dl issues

### ✅ System Stability
- No more system-wide network failures
- No manual restarts required
- Self-contained failure domain

### ✅ Security
- Download traffic isolated from main network
- VPN kill-switch protection maintained
- Controlled access via port forwarding

### ✅ Functionality
- External access maintained via port forwarding
- All services accessible from main network
- Internet connectivity for container via NAT

## Troubleshooting

### Container Can't Reach Internet
```bash
# Check NAT rule exists
iptables -t nat -L POSTROUTING | grep 192.168.100

# Check IP forwarding enabled
cat /proc/sys/net/ipv4/ip_forward
# Should return 1
```

### Port Forwarding Not Working
```bash
# Check DNAT rules
iptables -t nat -L PREROUTING | grep 192.168.100

# Check FORWARD rules
iptables -L FORWARD | grep 192.168.100

# Test direct access first
curl -I http://192.168.100.2:8080
```

### Bridge Not Working
```bash
# Check bridge status
ip addr show vmbr1
brctl show vmbr1

# Restart networking
systemctl restart networking
```

## Conclusion

This setup successfully isolates the media downloading stack while maintaining functionality and preventing system-wide failures. The isolated bridge approach provides a robust solution for running potentially unstable Docker containers in Proxmox without compromising the entire virtualization environment.