# Offline Meshtastic Map Station Setup Guide

A complete guide to setting up a Raspberry Pi 5 as an offline Meshtastic monitoring station with MeshMonitor and local OpenStreetMap tiles.

## Overview

This setup creates a self-contained base station that:
- Connects to your Meshtastic LoRa node (Heltec WiFi LoRa 32 V3)
- Displays real-time GPS positions of all mesh devices on a map
- Works completely offline with pre-downloaded map tiles
- Can be accessed via touchscreen, phone, or any device on the local network

## Hardware Requirements

- Raspberry Pi 5 (4GB recommended)
- MicroSD card (32GB+ recommended, Class 10)
- Power supply (USB-C, 5V 5A for Pi 5)
- Heltec WiFi LoRa 32 V3 (or similar Meshtastic node with WiFi)
- Optional: Touchscreen display (HDMI or DSI)

---

## Part 1: Get Started Now (No LoRa Hardware Needed)

These steps can be completed while waiting for your LoRa hardware to arrive. They're node-independent and will save you significant time later.

### 1.1 Flash Raspberry Pi OS

On your computer, download and use Raspberry Pi Imager:

```bash
# Download from: https://www.raspberrypi.com/software/
# Or on Ubuntu/Debian:
sudo apt install rpi-imager
```

Flash **Raspberry Pi OS Lite (64-bit)** to your SD card. In the imager settings (gear icon), configure:
- Hostname: `meshstation`
- Enable SSH (password authentication)
- Set username/password (e.g., `mesh` / `yourpassword`)
- Configure WiFi (for initial setup, can disable later)
- Set locale/timezone

### 1.2 First Boot and System Update

Insert SD card, power on the Pi, then SSH in:

```bash
ssh mesh@meshstation.local
# Or use the IP address if hostname doesn't resolve
```

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install epiphany-browser onboard at-spi2-core
sudo reboot
```

### 1.3 Switch from Wayland to X11

Onboard does not work properly under Wayland. You must switch to X11:

```bash
sudo raspi-config
```

Navigate to: **Advanced Options → Wayland → X11**

### 1.4 Enable Auto-Login

Navigate to: **System Options → Boot / Auto Login → Desktop Autologin**

This eliminates password prompts that would be difficult without a keyboard.

Reboot after changing:

```bash
sudo reboot
```

## On-Screen Keyboard Setup

### Configure Onboard

Start Onboard:

```bash
onboard &
```

Open Onboard **Preferences** and configure:

**General tab:**
- ✅ Enable "Show floating icon when Onboard is hidden"
- ✅ Enable "Auto-show when editing text" (keyboard appears automatically in text fields)

**Window tab:**
- ✅ Enable "Dock to screen edge"
- Set edge to "Bottom"
- ✅ Enable "Expand to landscape/portrait"
- ❌ Disable "Window decoration" (removes title bar with minimize/maximize/close buttons)
- ✅ Enable "Force to top" (keeps keyboard above other windows)

---

### Enable Auto-Start on Boot

Create the autostart entry:

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/onboard.desktop
```

Paste the following:

```ini
[Desktop Entry]
Type=Application
Exec=onboard
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Onboard
```

Save (Ctrl+X, Y, Enter) and reboot.

---


### 1.5 Install Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
```

Log out and back in for group change to take effect:

```bash
exit
```

SSH back in and verify Docker works:

```bash
docker --version
docker run hello-world
```

### 1.6 Download Your Offline Map Tiles

This is the most time-consuming part and requires no LoRa hardware.

Download pre-made regional tiles from MapTiler:
https://www.maptiler.com/on-prem-datasets/dataset/osm/#0.22/0/0

1. Create a free account
2. Download the tileset for your region (e.g., Quebec or a smaller area)
3. Transfer to your Pi

### 1.7 Set Up TileServer GL Light

You can run just the tile server portion to verify your maps work:

```bash
mkdir -p ~/meshmonitor/tiles
cd ~/meshmonitor

cat > docker-compose.yml << 'EOF'
services:
  tileserver:
    image: maptiler/tileserver-gl-light:latest
    container_name: tileserver
    ports:
      - "8081:8080"
    volumes:
      - ./tiles:/data
    restart: unless-stopped
EOF

docker compose up -d
```

Then browse to `http://meshstation.local:8081` to confirm your tiles render correctly.

---

## Part 2: Configure Your Meshtastic Node

*Continue here once your LoRa hardware arrives.*

Your Heltec LoRa node needs WiFi enabled so MeshMonitor can connect to it.

### 2.1 Connect to Your Meshtastic Node

Use the Meshtastic app on your phone or the Python CLI to configure WiFi:

```bash
# Install Meshtastic Python CLI (optional, can use phone app instead)
pip install meshtastic --break-system-packages

# Connect via USB and set WiFi credentials
meshtastic --set wifi.enabled true
meshtastic --set wifi.ssid "YourWiFiNetwork"
meshtastic --set wifi.psk "YourWiFiPassword"
```

### 2.2 Find Your Node's IP Address

After enabling WiFi, your node will get an IP address. Find it via:
- Your router's admin page (look for connected devices)
- The Meshtastic app (shows IP when connected via Bluetooth)
- Network scanning: `nmap -sn 192.168.1.0/24`

Note this IP address (e.g., `192.168.1.150`) - you'll need it for MeshMonitor.

### 2.3 Set Fixed Position for Base Station

Since your base station won't have GPS, set a fixed position:

```bash
# Set your chalet's coordinates (replace with actual lat/long)
meshtastic --setlat 45.5678 --setlon -73.1234 --setalt 150
```

---

## Part 3: Install MeshMonitor

### 3.1 Create Project Directory

If you already set up TileServer in Part 1, skip this step:

```bash
mkdir -p ~/meshmonitor
cd ~/meshmonitor
```

### 3.2 Create Docker Compose File

Update your docker-compose.yml to include MeshMonitor alongside the tile server:

```bash
cd ~/meshmonitor

cat > docker-compose.yml << 'EOF'
services:
  meshmonitor:
    image: ghcr.io/yeraze/meshmonitor:latest
    container_name: meshmonitor
    ports:
      - "8080:3001"
    volumes:
      - meshmonitor-data:/data
    environment:
      - MESHTASTIC_NODE_IP=192.168.1.150  # Change to your node's IP
    restart: unless-stopped
    depends_on:
      - tileserver

  tileserver:
    image: maptiler/tileserver-gl-light:latest
    container_name: tileserver
    ports:
      - "8081:8080"
    volumes:
      - ./tiles:/data
    restart: unless-stopped

volumes:
  meshmonitor-data:
EOF
```

**Important:** Replace `192.168.1.150` with your Meshtastic node's actual IP address.

### 3.3 Start MeshMonitor

```bash
docker compose down  # Stop existing services if running
docker compose up -d
```

Verify it's running:

```bash
docker compose logs -f
# Press Ctrl+C to exit logs
```

### 3.4 Access MeshMonitor

From any device on the same network, open a browser and go to:

```
http://meshstation.local:8080
```

Or use the Pi's IP address directly.

**Default login:** `admin` / `changeme`

**Change the password immediately** via Settings → User Management.

---

## Part 4: Additional Offline Map Options

If you need more control over your map tiles, here are alternative methods:

### 4.1 Method A: Using HOT Export Tool

This method downloads raster tiles in MBTiles format.

1. Go to https://export.hotosm.org/
2. Create an OpenStreetMap account if needed
3. Draw a polygon around your chalet area (include surrounding wilderness)
4. Select **MBTiles** as the export format
5. Choose zoom levels 8-14 (good balance of coverage and detail)
6. Submit and wait for the download link (you'll get an email)
7. Download the `.mbtiles` file

Transfer to your Pi:

```bash
# From your computer
scp ~/Downloads/your-export.mbtiles mesh@meshstation.local:~/meshmonitor/tiles/
```

### 4.2 Method B: Using tile-downloader

Install Python and download tiles directly:

```bash
# On the Pi
pip install tile-downloader --break-system-packages

# Download tiles for a specific bounding box
# Format: West, South, East, North (longitude, latitude)
# Example for an area in Quebec:
tile-downloader \
  --url "https://tile.openstreetmap.org/{z}/{x}/{y}.png" \
  --bbox "-74.5,45.0,-72.5,47.0" \
  --zoom "8-14" \
  --output ~/meshmonitor/tiles/quebec.mbtiles
```

**Note:** Be respectful of OSM tile usage policies. Download once and use locally.

---

## Part 5: Configure MeshMonitor for Offline Tiles

### 5.1 Configure MeshMonitor to Use Local Tiles

1. Open MeshMonitor in your browser: `http://meshstation.local:8080`
2. Go to **Settings** → **Map Settings** (or similar)
3. Add a custom tile server:
   - **Name:** Local OSM
   - **URL:** `http://meshstation.local:8081/styles/osm-bright/{z}/{x}/{y}.png`
   - Or for raw tiles: `http://meshstation.local:8081/data/v3/{z}/{x}/{y}.pbf`
4. Save and refresh the map

### 5.2 Alternative: Direct XYZ Tile Directory

If your tiles are in standard XYZ directory format (z/x/y.png), you can serve them with nginx:

```bash
# Add nginx to docker-compose.yml
cat >> docker-compose.yml << 'EOF'

  nginx-tiles:
    image: nginx:alpine
    container_name: nginx-tiles
    ports:
      - "8082:80"
    volumes:
      - ./tiles:/usr/share/nginx/html/tiles:ro
    restart: unless-stopped
EOF

docker compose up -d
```

Then configure MeshMonitor to use: `http://meshstation.local:8082/tiles/{z}/{x}/{y}.png`

---

## Part 6: Optional - Touchscreen Kiosk Mode

If you have a touchscreen attached and want the Pi to boot directly into the map interface:

### 6.1 Install Minimal GUI Components

```bash
sudo apt install -y --no-install-recommends \
  xserver-xorg \
  x11-xserver-utils \
  xinit \
  openbox \
  chromium-browser \
  unclutter
```

### 6.2 Create Kiosk Startup Script

```bash
cat > ~/kiosk.sh << 'EOF'
#!/bin/bash

# Disable screen blanking
xset s off
xset s noblank
xset -dpms

# Hide mouse cursor after 0.5 seconds of inactivity
unclutter -idle 0.5 -root &

# Wait for MeshMonitor to be ready
sleep 10

# Start Chromium in kiosk mode
chromium-browser \
  --kiosk \
  --noerrdialogs \
  --disable-infobars \
  --disable-session-crashed-bubble \
  --check-for-update-interval=604800 \
  --disable-pinch \
  --overscroll-history-navigation=0 \
  http://localhost:8080
EOF

chmod +x ~/kiosk.sh
```

### 6.3 Configure Auto-Start

```bash
# Create .xinitrc
echo "exec openbox-session" > ~/.xinitrc

# Create openbox autostart
mkdir -p ~/.config/openbox
echo "~/kiosk.sh &" > ~/.config/openbox/autostart

# Add to .bash_profile to start X on login
echo '[[ -z $DISPLAY && $XDG_VTNR -eq 1 ]] && startx' >> ~/.bash_profile
```

### 6.4 Enable Auto-Login

```bash
sudo raspi-config
# Navigate to: System Options → Boot / Auto Login → Console Autologin
```

Reboot to test:

```bash
sudo reboot
```

---

## Part 7: Network Configuration for Field Use

### 7.1 Create a Local Access Point (Optional)

If you want devices to connect directly to the Pi without existing WiFi:

```bash
sudo apt install -y hostapd dnsmasq

# This is complex - consider using a dedicated router instead
# Or use the Pi as a WiFi client and let devices connect to the same network
```

### 7.2 Static IP Configuration (Recommended)

For reliable operation, give your Pi and Meshtastic node static IPs:

```bash
sudo nmcli con mod "Wired connection 1" ipv4.addresses 192.168.1.100/24
sudo nmcli con mod "Wired connection 1" ipv4.gateway 192.168.1.1
sudo nmcli con mod "Wired connection 1" ipv4.dns "192.168.1.1"
sudo nmcli con mod "Wired connection 1" ipv4.method manual
```

---

## Maintenance Commands

### View Logs

```bash
cd ~/meshmonitor
docker compose logs -f meshmonitor
docker compose logs -f tileserver
```

### Restart Services

```bash
cd ~/meshmonitor
docker compose restart
```

### Update MeshMonitor

```bash
cd ~/meshmonitor
docker compose pull
docker compose up -d
```

### Backup Data

```bash
# Backup MeshMonitor database
docker run --rm -v meshmonitor_meshmonitor-data:/data -v ~/backups:/backup \
  alpine tar czf /backup/meshmonitor-backup-$(date +%Y%m%d).tar.gz /data
```

---

## Troubleshooting

### MeshMonitor can't connect to Meshtastic node

1. Verify the node's IP address is correct
2. Ensure both devices are on the same network
3. Check that the Meshtastic node has WiFi enabled
4. Try pinging the node: `ping 192.168.1.150`

### Map tiles not loading

1. Verify TileServer is running: `docker compose ps`
2. Check tile server directly: `http://meshstation.local:8081`
3. Ensure tiles are in the correct directory
4. Check MeshMonitor tile server URL configuration

### Can't access MeshMonitor from other devices

1. Check Pi's firewall: `sudo ufw status`
2. Verify the service is running: `docker compose ps`
3. Try accessing via IP instead of hostname

### Performance issues

1. Reduce tile zoom levels (8-12 instead of 8-14)
2. Use raster tiles instead of vector tiles
3. Limit the geographic area of downloaded tiles

---

## Quick Reference

| Service | URL | Port |
|---------|-----|------|
| MeshMonitor | http://meshstation.local:8080 | 8080 |
| TileServer | http://meshstation.local:8081 | 8081 |
| SSH | meshstation.local | 22 |

| Default Credentials | |
|---------------------|--|
| MeshMonitor | admin / changeme |
| Pi SSH | mesh / (your password) |

---

## Summary

You now have:
- A Raspberry Pi 5 running MeshMonitor
- Connection to your Meshtastic LoRa base node
- Offline OpenStreetMap tiles for your region
- A web interface accessible from any device on the network
- Optional touchscreen kiosk mode for dedicated display

Your field teams with T-Echo devices will appear on the map as they move, and you can exchange messages through the Meshtastic mesh network—all without internet connectivity.
