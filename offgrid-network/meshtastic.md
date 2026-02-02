# Meshtastic Node Setup Guide

A guide to configuring your Heltec LoRa V3/V4 node for use with MeshMonitor on a standalone off-grid system.

## Overview

This guide covers:
- Flashing/recovering Meshtastic firmware
- Connecting the node to your Raspberry Pi via USB
- Configuring the node for Canadian operation (915MHz)
- Setting up for offline/standalone use (no home network required)

## Hardware

- Heltec WiFi LoRa 32 V3 or V4
- USB-A to USB-C cable
- Raspberry Pi 5 (or computer for initial flashing)

---

## Part 1: Flash Meshtastic Firmware

If your node is new, stuck in a boot loop, or showing "LoRa Error", you'll need to flash the firmware.

### 1.1 Enter Bootloader Mode

If the device is boot-looping or unresponsive:

1. **Hold the BOOT button** (small button, sometimes labeled "PRG" or "0")
2. **While holding BOOT, press and release the RESET button** (labeled "RST")
3. **Release the BOOT button**

The screen should go blank - the device is now in bootloader mode, waiting for firmware.

### 1.2 Flash via Web Flasher

1. Open https://flasher.meshtastic.org in **Chrome or Edge** (Firefox doesn't support WebSerial)

2. Click **Connect** and select your device's serial port

3. Select:
   - **Device:** Heltec V3 (or V4 if listed)
   - **Version:** Latest stable

4. Select baud rate: **921600** (or **115200** if flashing fails)

5. Click **"Full Erase and Install"**

6. Wait for completion - you'll see progress like:
   ```
   Erasing flash (this may take a while)...
   Chip erase completed successfully
   Writing at 0x0... (1%)
   ...
   Writing at 0x2037a9... (100%)
   Hard resetting via RTS pin...
   ```

7. Press the **RESET button** on the device after flashing completes

The device should now boot into Meshtastic with a welcome screen.

### 1.3 Recovery: If Device Won't Enter Bootloader

If the web flasher can't detect the device, use esptool directly:

```bash
# Install esptool on your computer
pip install esptool

# With device in bootloader mode, erase flash completely
esptool.py --chip esp32s3 --port /dev/ttyUSB0 erase_flash
```

Then retry the web flasher.

---

## Part 2: Connect to Raspberry Pi

For a standalone off-grid system, connect the Meshtastic node directly to the Pi via USB. This is more reliable than WiFi and doesn't require a home network.

### 2.1 Physical Connection

Connect the Heltec to the Raspberry Pi using a USB-A to USB-C cable.

### 2.2 Find the Serial Port

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

You'll see something like `/dev/ttyACM0` or `/dev/ttyUSB0`.

To confirm it's your Heltec:

```bash
lsusb
```

Look for "Espressif USB JTAG/serial debug unit" - that's your device.

### 2.3 Install Meshtastic CLI

```bash
pip install meshtastic --break-system-packages
```

Add the install location to your PATH:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 2.4 Test Connection

```bash
meshtastic --port /dev/ttyACM0 --info
```

You should see device information including firmware version, node ID, and current settings.

---

## Part 3: Configure the Node

### 3.1 Set Region (Required)

**Critical:** The node won't transmit until you set the region. For Canada and USA, use the US region (915MHz ISM band):

```bash
meshtastic --port /dev/ttyACM0 --set lora.region US
```

### 3.2 Set Node Name

Give your base station a friendly name:

```bash
meshtastic --port /dev/ttyACM0 --set-owner "Base Station"
```

You can also set a short name (4 characters max):

```bash
meshtastic --port /dev/ttyACM0 --set-owner-short "BASE"
```

### 3.3 Set Fixed Position (Base Station)

Since your base station is stationary, set a fixed GPS position:

```bash
# Replace with your actual coordinates
meshtastic --port /dev/ttyACM0 --setlat 45.5678 --setlon -73.1234 --setalt 150
```

To find your coordinates:
- Use Google Maps (right-click → "What's here?")
- Use your phone's GPS app
- Use https://www.latlong.net/

### 3.4 Verify Configuration

```bash
meshtastic --port /dev/ttyACM0 --info
```

Check that:
- `"region": "US"` (not "UNSET")
- Owner name is set
- Position shows your coordinates

---

## Part 4: Connection Options for MeshMonitor

### Option A: USB Serial (Recommended for Base Station)

The simplest and most reliable method. The node connects directly to the Pi via USB cable.

**Pros:**
- No wireless configuration needed
- Most reliable connection
- Lower power consumption

**Cons:**
- Physical cable required
- Node must be inside or close to the case

MeshMonitor can connect via serial port at `/dev/ttyACM0`.

### Option B: Pi as WiFi Access Point

The Pi hosts its own WiFi network, and the Heltec connects to it. Useful if you want other devices to also connect.

**Pros:**
- Wireless flexibility
- Other devices can connect to the same network

**Cons:**
- More complex setup
- Slightly higher power draw

### Option C: Bluetooth

Connect the Pi to the Heltec via Bluetooth.

**Pros:**
- No cables, no WiFi configuration

**Cons:**
- MeshMonitor's Bluetooth support is less mature

---

## Troubleshooting

### "LoRa Error" on Screen

The region isn't set. Run:
```bash
meshtastic --port /dev/ttyACM0 --set lora.region US
```

### Boot Loop (Continuous Restart)

Firmware is corrupted. Follow Part 1 to re-flash:
1. Enter bootloader mode (hold BOOT, press RESET)
2. Use web flasher with "Full Erase and Install"

### "meshtastic: command not found"

The CLI isn't in your PATH. Fix with:
```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### Device Not Detected

Check if the device appears:
```bash
lsusb
dmesg | tail -20
```

Try a different USB cable or port. Some cables are charge-only and don't support data.

### White LED Flashing

This is normal - indicates the radio is active and listening.

---

## Quick Reference

### Common Commands

```bash
# View all device info
meshtastic --port /dev/ttyACM0 --info

# Set region (required for transmission)
meshtastic --port /dev/ttyACM0 --set lora.region US

# Set owner name
meshtastic --port /dev/ttyACM0 --set-owner "My Node"

# Set fixed position
meshtastic --port /dev/ttyACM0 --setlat 45.5678 --setlon -73.1234 --setalt 150

# Factory reset
meshtastic --port /dev/ttyACM0 --factory-reset

# Reboot device
meshtastic --port /dev/ttyACM0 --reboot
```

### Serial Port Locations

| Device | Typical Port |
|--------|--------------|
| Heltec V3/V4 | `/dev/ttyACM0` |
| Older ESP32 boards | `/dev/ttyUSB0` |

### LoRa Regions

| Region | Frequency | Use In |
|--------|-----------|--------|
| US | 915MHz | USA, Canada, South America |
| EU_868 | 868MHz | Europe, UK |
| AU_915 | 915MHz | Australia, New Zealand |
| CN | 470MHz | China |

---

## Next Steps

Once your node is configured:
1. Connect it to MeshMonitor (see MeshMonitor setup guide)
2. Configure additional portable nodes (T-Echo, T-Deck) the same way
3. Test range and coverage in your area
