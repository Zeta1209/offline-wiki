# MeshMonitor Setup Guide

A guide to installing and configuring MeshMonitor with your Meshtastic node and offline map tiles for a completely standalone off-grid system.

## Overview

This guide covers:
- Connecting MeshMonitor to your Meshtastic node via USB serial
- Configuring the Serial Bridge for USB communication
- Setting up offline map tiles with TileServer GL Light
- Configuring MeshMonitor to use local tiles

## Prerequisites

Before starting, ensure you have:
- Raspberry Pi 5 with Docker installed (see Part 1 of the main setup guide)
- Meshtastic node configured and connected via USB (see Meshtastic Node Setup Guide)
- Offline map tiles downloaded (see Part 1 of the main setup guide)

---

## Part 1: Prepare Your Meshtastic Node for Serial Communication

Before using the Serial Bridge, configure your device for serial communication.

### 1.1 Enable Serial Mode

Connect to your node and configure serial settings:

```bash
# Enable serial mode on the device
meshtastic --port /dev/ttyACM0 --set serial.enabled true
meshtastic --port /dev/ttyACM0 --set serial.echo false
meshtastic --port /dev/ttyACM0 --set serial.mode SIMPLE
meshtastic --port /dev/ttyACM0 --set serial.baud BAUD_115200
```

### 1.2 Verify Serial Settings

```bash
meshtastic --port /dev/ttyACM0 --get serial
```

You should see:
```
serial.enabled: True
serial.echo: False
serial.mode: SIMPLE
serial.baud: BAUD_115200
```

### 1.3 Add User to dialout Group

Ensure your user can access serial devices:

```bash
sudo usermod -aG dialout $USER
```

Log out and back in for the change to take effect.

---

## Part 2: Create the Docker Compose Configuration

### 2.1 Create Project Directory

```bash
mkdir -p ~/meshmonitor/tiles
cd ~/meshmonitor
```

### 2.2 Create Docker Compose File

Create a `docker-compose.yml` with the Serial Bridge, MeshMonitor, and TileServer:

```bash
cat > docker-compose.yml << 'EOF'
services:
  serial-bridge:
    image: ghcr.io/yeraze/meshtastic-serial-bridge:latest
    container_name: meshtastic-serial-bridge
    devices:
      - /dev/ttyACM0:/dev/ttyACM0
    ports:
      - "4403:4403"
    restart: unless-stopped
    environment:
      - SERIAL_DEVICE=/dev/ttyACM0
      - BAUD_RATE=115200
      - TCP_PORT=4403

  meshmonitor:
    image: ghcr.io/yeraze/meshmonitor:latest
    container_name: meshmonitor
    ports:
      - "8080:3001"
    volumes:
      - meshmonitor-data:/data
    environment:
      - MESHTASTIC_NODE_IP=serial-bridge
      - ALLOWED_ORIGINS=http://localhost:8080,http://offgrid.local:8080
    restart: unless-stopped
    depends_on:
      - serial-bridge

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

**Important notes:**
- Replace `/dev/ttyACM0` with your actual device path if different
- Update `ALLOWED_ORIGINS` to include your Pi's hostname or IP address
- The `devices` section passes the USB device into the container

### 2.3 Verify Your Device Path

Before starting, confirm your Meshtastic node is connected:

```bash
ls /dev/ttyACM* /dev/ttyUSB*
```

Update the docker-compose.yml if your device is at a different path (e.g., `/dev/ttyUSB0`).

---

## Part 3: Start the Services

### 3.1 Start Everything

```bash
cd ~/meshmonitor
docker compose up -d
```

### 3.2 Verify Services Are Running

```bash
docker compose ps
```

You should see all three containers running:
```
NAME                        STATUS
meshtastic-serial-bridge    Up
meshmonitor                 Up
tileserver                  Up
```

### 3.3 Check the Logs

Check the Serial Bridge connection:

```bash
docker compose logs serial-bridge
```

Look for:
- "Device /dev/ttyACM0 is ready"
- "socat listening on 0.0.0.0:4403"

Check MeshMonitor:

```bash
docker compose logs meshmonitor
```

Look for:
- "Connected to Meshtastic node at serial-bridge:4403"

### 3.4 Test Serial Bridge Connection

From the Pi, test that the bridge is working:

```bash
meshtastic --host localhost --info
```

This should return your node's information.

---

## Part 4: Access MeshMonitor

### 4.1 Open the Web Interface

From any device on the same network, open a browser and go to:

```
http://offgrid.local:8080
```

Or use the Pi's IP address:

```
http://192.168.x.x:8080
```

### 4.2 Login

**Default credentials:**
- Username: `admin`
- Password: `changeme`

**Change the password immediately** via Settings → User Management.

### 4.3 Verify Node Connection

Once logged in, you should see:
- Your base station node in the node list
- Connection status showing "Connected"
- Node information (name, battery level, etc.)

---

## Part 5: Configure Offline Map Tiles

### 5.1 Verify TileServer Is Running

Open the TileServer interface in a browser:

```
http://offgrid.local:8081
```

You should see a list of available tile styles based on your downloaded tiles.

### 5.2 Find Your Tile URL

TileServer GL Light provides tiles in different formats. Common URLs:

**For vector tiles (.pbf):**
```
http://offgrid.local:8081/data/v3/{z}/{x}/{y}.pbf
```

**For raster tiles (.png):**
```
http://offgrid.local:8081/styles/osm-bright/{z}/{x}/{y}.png
```

The exact URL depends on your tile data. Check the TileServer web interface at port 8081 to see available styles and data endpoints.

### 5.3 Add Custom Tile Server in MeshMonitor

1. Open MeshMonitor: `http://offgrid.local:8080`
2. Go to **Settings** → **Map Settings** → **Custom Tile Servers**
3. Click **+ Add Custom Tile Server**
4. Configure:

**For vector tiles:**
```
Name: Local Vector Tiles
URL: http://offgrid.local:8081/data/v3/{z}/{x}/{y}.pbf
Attribution: © OpenStreetMap contributors
Max Zoom: 14
Description: Offline vector map tiles
```

**For raster tiles:**
```
Name: Local OSM
URL: http://offgrid.local:8081/styles/osm-bright/{z}/{x}/{y}.png
Attribution: © OpenStreetMap contributors
Max Zoom: 18
Description: Offline OpenStreetMap tiles
```

5. Click **Save**
6. Select your custom tileset from the map tileset dropdown

### 5.4 Verify Offline Maps Work

1. Disconnect from the internet (disable WiFi/Ethernet temporarily)
2. Refresh MeshMonitor
3. The map should still load and display your local tiles
4. Zoom and pan to verify tiles are available for your area

---

## Part 6: Configure Base Station Position

If your base station doesn't have GPS, set a fixed position so it appears on the map.

### 6.1 Set Fixed Position via CLI

```bash
# Replace with your actual coordinates
meshtastic --port /dev/ttyACM0 --setlat 45.5678 --setlon -73.1234 --setalt 150
```

### 6.2 Set Fixed Position via MeshMonitor

1. In MeshMonitor, go to the **Device** tab
2. Find **Position Settings**
3. Enable **Fixed Position**
4. Enter your latitude, longitude, and altitude
5. Click **Save**

The device will reboot and your base station will appear on the map.

---

## Part 7: Maintenance Commands

### View Logs

```bash
cd ~/meshmonitor

# All services
docker compose logs -f

# Specific service
docker compose logs -f meshmonitor
docker compose logs -f serial-bridge
docker compose logs -f tileserver
```

### Restart Services

```bash
cd ~/meshmonitor
docker compose restart
```

### Stop Services

```bash
cd ~/meshmonitor
docker compose down
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
docker run --rm \
  -v meshmonitor_meshmonitor-data:/data \
  -v ~/backups:/backup \
  alpine tar czf /backup/meshmonitor-backup-$(date +%Y%m%d).tar.gz /data
```

---

## Troubleshooting

### Serial Bridge Can't Find Device

**Symptoms:** Serial bridge logs show "Device not found"

**Solutions:**

1. Verify device is connected:
   ```bash
   ls /dev/ttyACM* /dev/ttyUSB*
   ```

2. Check system logs:
   ```bash
   dmesg | grep tty
   ```

3. Update device path in docker-compose.yml if it changed

4. Ensure user is in dialout group:
   ```bash
   groups $USER  # Should include 'dialout'
   ```

### MeshMonitor Shows "Connection Failed"

**Solutions:**

1. Check serial bridge is running:
   ```bash
   docker compose ps serial-bridge
   ```

2. Test bridge directly:
   ```bash
   meshtastic --host localhost --info
   ```

3. Check bridge logs for errors:
   ```bash
   docker compose logs serial-bridge
   ```

4. Verify serial settings on device:
   ```bash
   meshtastic --port /dev/ttyACM0 --get serial
   ```

### Map Tiles Not Loading

**Solutions:**

1. Verify TileServer is running:
   ```bash
   docker compose ps tileserver
   ```

2. Check TileServer directly:
   ```bash
   curl http://localhost:8081/
   ```

3. Verify tiles exist:
   ```bash
   ls -la ~/meshmonitor/tiles/
   ```

4. Check the tile URL format matches your data

### Device Keeps Disconnecting

**Solutions:**

1. Check USB cable - use a data cable, not charge-only
2. Try a different USB port
3. Check power supply is adequate
4. Review serial bridge logs for patterns

### CORS Errors / Blank Page

**Solutions:**

1. Verify `ALLOWED_ORIGINS` in docker-compose.yml includes your access URL
2. Add multiple origins if accessing from different addresses:
   ```yaml
   - ALLOWED_ORIGINS=http://localhost:8080,http://offgrid.local:8080,http://192.168.1.100:8080
   ```
3. Restart MeshMonitor after changing:
   ```bash
   docker compose restart meshmonitor
   ```

---

## Quick Reference

### Service URLs

| Service | URL | Port |
|---------|-----|------|
| MeshMonitor | http://offgrid.local:8080 | 8080 |
| TileServer | http://offgrid.local:8081 | 8081 |
| Serial Bridge (internal) | serial-bridge:4403 | 4403 |

### Default Credentials

| Service | Username | Password |
|---------|----------|----------|
| MeshMonitor | admin | changeme |

### Common Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# View logs
docker compose logs -f

# Restart a service
docker compose restart meshmonitor

# Update images
docker compose pull && docker compose up -d

# Check device connection
meshtastic --host localhost --info
```

### File Locations

| Item | Location |
|------|----------|
| Docker Compose | ~/meshmonitor/docker-compose.yml |
| Map Tiles | ~/meshmonitor/tiles/ |
| MeshMonitor Data | Docker volume: meshmonitor-data |

---

## Next Steps

Once MeshMonitor is running:
1. Configure additional portable nodes (T-Echo, T-Deck) - they'll appear automatically when in range
2. Set up channels for your mesh network
3. Test coverage and range in your area
4. Consider setting up the Pi as a WiFi access point for field access (optional)
