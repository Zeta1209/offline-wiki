# Offline Wikipedia Project on Raspberry Pi Zero W

## Objective

Create an autonomous Wi-Fi access point serving offline Wikipedia via Kiwix:

- **Built-in Wi-Fi (wlan0)** → Access Point "PiWiki"
- **USB Wi-Fi Dongle (wlan1)** → Connection to home network (LAN/Internet)
- **Kiwix** → HTTP server for offline Wikipedia

---

## Required Hardware

| Component | Notes |
|-----------|-------|
| Raspberry Pi Zero / Zero W | With built-in Wi-Fi |
| microSD Card | 512 GB recommended for ZIM files |
| USB Wi-Fi Dongle | For LAN connection |
| Powered USB Hub | Avoids power issues |
| Power Cable | USB-C or micro-USB depending on model |
| Mini-HDMI Adapter | For initial setup (optional) |

---

## Step 0 — Prepare the SD Card

1. Download **Raspberry Pi OS Lite (32-bit)**
2. Flash with **Raspberry Pi Imager**
3. In Imager settings, configure:
   - Hostname (e.g., `piwiki`)
   - SSH enabled
   - Wi-Fi credentials for your home network
   - Username and password
4. Insert the card and boot the Pi

---

## Step 1 — Initial SSH Connection

```bash
# Find the Pi's IP (from your router or network scan)
# Connect
ssh piwiki@<PI_IP_ADDRESS>
```
```bash
# Change default password if needed
passwd
```
```bash
# Update your pi (this might take long, remember youre on a pi with about 512MB or RAM)
sudo apt update
sudo apt full-upgrade -y
```
```bash
# Well reboot ofc
sudo reboot
```

---

## Step 2 — Prepare Kiwix and ZIM Files

### Create storage directory

```bash
sudo mkdir -p /data/zim
sudo chown -R piwiki:piwiki /data
```

### Transfer ZIM files (from your PC)

Use WinSCP, scp, or rsync to copy ZIM files to `/data/zim/` IMPORTANT, for now only transfer one or two ZIM files for test purposes, if you lose all access to your PI during the configuration you will need to re-flash your micro-SD card and restart from scratch.

### Install Kiwix

```bash
sudo apt update
sudo apt install -y kiwix-tools
```

### Test Kiwix manually

```bash
kiwix-serve --port=8080 /data/zim/*.zim
```

---

## Step 3 — Systemd Service for Kiwix

### Create service file

```bash
sudo nano /etc/systemd/system/kiwix.service
```

**Contents:**

```ini
[Unit]
Description=Kiwix Offline Wikipedia
After=network.target

[Service]
User=piwiki
ExecStart=/bin/bash -c '/usr/bin/kiwix-serve --port=8080 /data/zim/*.zim'
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

### Enable the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable kiwix
sudo systemctl start kiwix
sudo systemctl status kiwix --no-pager
```

---

## Step 4 — Configure the USB Dongle (Home Network Connection)

When you configured Raspberry Pi OS with Imager, both Wi-Fi interfaces (built-in and USB dongle) will automatically connect to your home network via NetworkManager.

### Verify both interfaces are connected

```bash
# Check network status
nmcli device status

# You should see:
# wlan0  wifi  connected  <your-network>
# wlan1  wifi  connected  <your-network>

# Check IPs
ip addr
```

### Note the USB dongle IP

```bash
ip addr show wlan1 | grep "inet "
```

**Important:** Write down this IP (e.g., `10.0.0.158`) — this is your backup SSH access once wlan0 becomes an AP.

---

## Step 5 — Configure the Access Point (Built-in Wi-Fi)

### 5.1 — Connect via USB dongle IP

Before proceeding, SSH into your Pi using the **wlan1 IP address** to avoid losing connection:

```bash
ssh piwiki@<WLAN1_IP>
```

### 5.2 — Tell NetworkManager to ignore wlan0

```bash
# Disconnect wlan0 from home network
sudo nmcli connection down <your-network-connection-name>

# Tell NetworkManager to stop managing wlan0
sudo nmcli device set wlan0 managed no

# Verify wlan0 is unmanaged
nmcli device status
# wlan0 should show "unmanaged"
```

### 5.3 — Make NetworkManager ignore wlan0 permanently

```bash
sudo nano /etc/NetworkManager/conf.d/ignore-wlan0.conf
```

**Contents:**

```
[keyfile]
unmanaged-devices=interface-name:wlan0
```

### 5.4 — Install hostapd and dnsmasq

```bash
sudo apt update
sudo apt install -y hostapd dnsmasq
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
```

### 5.5 — Configure static IP for wlan0

```bash
sudo nano /etc/systemd/network/10-wlan0-ap.network
```

**Contents:**

```
[Match]
Name=wlan0

[Network]
Address=192.168.50.1/24
ConfigureWithoutCarrier=yes
```

Enable systemd-networkd:

```bash
sudo systemctl enable systemd-networkd
```

### 5.6 — Configure hostapd

```bash
sudo nano /etc/hostapd/hostapd.conf
```

**Contents:**

```
interface=wlan0
driver=nl80211
ssid=PiWiki
hw_mode=g
channel=7
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=piwikioffline
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

**Point hostapd to the config file:**

```bash
sudo nano /etc/default/hostapd
```

Find `#DAEMON_CONF=""` and replace with:

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### 5.7 — Configure dnsmasq

```bash
# Backup original config
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig

# Create new config
sudo nano /etc/dnsmasq.conf
```

**Contents:**

```
interface=wlan0
dhcp-range=192.168.50.10,192.168.50.100,255.255.255.0,24h
```

### 5.8 — Enable and start services

```bash
sudo systemctl unmask hostapd
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```

### 5.9 — Verify services are running

```bash
sudo systemctl status hostapd --no-pager
sudo systemctl status dnsmasq --no-pager
ip addr show wlan0
```

You should see:
- hostapd: `active (running)`, `AP-ENABLED`
- dnsmasq: `active (running)`
- wlan0: `inet 192.168.50.1/24`

---

## Step 6 — Test and Reboot

### Test the Access Point

1. On your phone/laptop, look for Wi-Fi network **"PiWiki"**
2. Connect with password: **piwikioffline**
3. Open browser and go to: `http://192.168.50.1:8080`
4. You should see the Kiwix library!

### Reboot and verify persistence

```bash
sudo reboot
```

After reboot:
1. SSH via USB dongle IP: `ssh piwiki@<WLAN1_IP>`
2. Verify services: `systemctl status hostapd dnsmasq kiwix`
3. Test AP connection from phone

---

## Final Result

| Interface | Role | Network |
|-----------|------|---------|
| wlan0 | Access Point | PiWiki (192.168.50.1) |
| wlan1 | Wi-Fi Client | Home network |

### Access offline Wikipedia

Connect to Wi-Fi **PiWiki** then open:

```
http://192.168.50.1:8080
```

### SSH access from home network

```bash
ssh piwiki@<WLAN1_IP>
```

### SSH access from AP network

```bash
ssh piwiki@192.168.50.1
```

---

## Usage Scenarios

### With USB dongle connected (at home)
- SSH via home network (wlan1 IP)
- Internet access on Pi for updates/downloads
- AP still works for Kiwix access

### Without USB dongle (portable/offline)
- AP works independently
- SSH via AP network (192.168.50.1)
- No internet on Pi
- Perfect for offline/survival use

---

## Adding ZIM Files

Copy new ZIM files to `/data/zim/` and restart Kiwix:

```bash
sudo systemctl restart kiwix
```

### Recommended ZIM files for survival/offline use

| ZIM | Content |
|-----|---------|
| `wikipedia_en_all_maxi` | Full English Wikipedia |
| `wikimed_en_all_maxi` | Medical Wikipedia |
| `wikibooks_en_all_maxi` | Practical guides |
| `ifixit_en_all` | Repair guides |
| `wikem_en_all_maxi` | Emergency medicine |

Download from: https://download.kiwix.org/zim/

---

## Troubleshooting

### Access point doesn't appear

```bash
sudo systemctl status hostapd
journalctl -u hostapd -e
```

### Kiwix doesn't start

```bash
sudo systemctl status kiwix
ls -la /data/zim/
```

### No internet via USB dongle

```bash
nmcli device status
ip addr show wlan1
ping -I wlan1 8.8.8.8
```

### Lost SSH access

If you lose access via wlan1, connect to the PiWiki AP and SSH to 192.168.50.1

### Services not starting after reboot

```bash
sudo systemctl enable hostapd dnsmasq kiwix systemd-networkd
sudo reboot
```

---

## Important Notes

- **Pi Zero W cannot do AP + client on the same interface** — USB dongle is required for both functions simultaneously
- **Powered USB hub recommended** — avoids power issues with the dongle
- **NetworkManager handles wlan1** — no manual wpa_supplicant configuration needed
- **systemd-networkd handles wlan0** — provides static IP for the AP
- **hostapd handles AP mode** — more reliable than NetworkManager for AP on Pi Zero
