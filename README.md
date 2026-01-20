# Gridless

**Self-reliant communication and information systems for off-grid living.**

---

## Project Vision

Gridless is a personal project to build technology infrastructure that works independently of traditional networks — no internet, no cell towers, no reliance on external services. Whether at a remote chalet in northern Quebec or during an emergency, these systems provide:

- **Offline knowledge access** — Wikipedia, medical references, repair guides
- **Local mesh communication** — Encrypted messaging and GPS sharing
- **Offline navigation** — OpenStreetMap tiles for the region
- **Portable, rugged, solar-powered** designs

---

## Current Systems

### 1. PiWiki — Offline Wikipedia Server

| | |
|---|---|
| **Status** | ✅ Operational |
| **Hardware** | Raspberry Pi Zero W + USB WiFi dongle |
| **Storage** | 512GB microSD with ZIM files |

A Raspberry Pi Zero W configured as a standalone WiFi access point serving offline Wikipedia and other reference materials via Kiwix.

**Features:**
- Dual-network setup (AP on wlan0, home network on wlan1)
- Serves Wikipedia, WikiMed, iFixit, Wikibooks
- Fully portable — works without any internet connection
- SSH accessible from both networks

**Access:** Connect to WiFi `PiWiki` → http://192.168.50.1

📄 Documentation: [piwiki-setup.md](./piwiki/README.md)

---

### 2. Gridless Network Box — Off-Grid Mesh Communication

| | |
|---|---|
| **Status** | 🚧 Planning / Parts Acquisition |
| **Hardware** | Pelican case, Raspberry Pi 5, LoRa radios, LiFePO4 battery |

A rugged, waterproof pelican-style case containing a complete off-grid communication and information system for ~10 users.

**Planned Features:**
- LoRa mesh network (Meshtastic) with AES256 encryption
- Text messaging between users without internet
- GPS location sharing across the mesh
- Offline OpenStreetMap tiles for Quebec
- Offline Wikipedia (expanded from PiWiki)
- WiFi access point for local devices
- 24-40 hour battery runtime
- Solar recharging capability
- Waterproof controls and detachable antenna
- 3D printed internal component mounts

📄 Documentation: [gridless-network.md](./offgrid-network/README.md)
