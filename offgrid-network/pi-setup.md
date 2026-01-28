# Raspberry Pi Touchscreen Setup Guide

A complete guide for setting up a Raspberry Pi with automatic GUI boot and touch-friendly on-screen keyboard — no physical keyboard required after initial setup.

## Overview

This setup provides:

- Desktop that auto-starts at boot
- Touchscreen functions as mouse input
- Floating on-screen keyboard button (iPhone AssistiveTouch style)
- Power-cycle safe — no manual commands needed
- Works headlessly via SSH when needed

**Target use case:** Touchscreen-only Pi installations where you may not always have a keyboard attached.

---

## Requirements

- Raspberry Pi (3/4/5 recommended)
- Compatible touchscreen display
- SSH access or temporary keyboard for initial setup

---

## Installation Steps

### 1. Install Raspberry Pi OS Lite

Start with Lite to avoid unnecessary defaults and keep the system lean.

### 2. Install Desktop Environment and Display Manager

```bash
sudo apt update
sudo apt install raspberrypi-ui-mods lightdm
```

This installs:
- LXDE desktop
- Panel and window manager
- LightDM display manager

### 3. Enable Automatic GUI Start

```bash
sudo systemctl set-default graphical.target
sudo systemctl enable lightdm
```

The desktop will now appear automatically on boot — no keyboard required.

### 4. Configure Display Reliability

Edit the boot configuration:

```bash
sudo nano /boot/config.txt
```

Add the following line:

```
hdmi_force_hotplug=1
```

This ensures the screen works even if powered on slightly after the Pi boots.

### 5. Enable Auto-Login

```bash
sudo raspi-config
```

Navigate to: **System Options → Boot / Auto Login → Desktop Autologin**

This eliminates password prompts that would be difficult without a keyboard.

---

## On-Screen Keyboard Setup

### Install Onboard

Onboard is the best on-screen keyboard for touch interfaces on Raspberry Pi.

```bash
sudo apt install onboard
```

### Configure the Floating Button

Start Onboard once (via SSH or temporary keyboard):

```bash
onboard &
```

Open Onboard Preferences and configure:

**General tab:**
- ✅ Enable "Show floating icon"
- ❌ Disable "Auto-show when editing text" (optional, can be annoying)

**Window tab:**
- Set Docking to "Floating"
- ✅ Enable "Keep window on top"

**Recommended polish:**
- ✅ Enable "Lock position" to prevent accidental moves
- Set opacity to 30–40% for a subtle but visible button

### Enable Auto-Start on Boot

Create the autostart directory and desktop entry:

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

Save and reboot. The floating keyboard button will now appear automatically.

---

## How It Works

After setup, every boot will:

1. Start the desktop automatically
2. Display a small semi-transparent floating button
3. Tap the button → keyboard slides in
4. Tap again → keyboard hides
5. Button always stays accessible and on top

---

## Resource Usage

Running the GUI does consume some resources, but it's a worthwhile tradeoff for boot safety:

| Metric | Typical Value |
|--------|---------------|
| Idle RAM | ~300–400 MB |
| CPU | Nearly idle |

This is acceptable on Pi 3/4/5 and prevents "bricked-until-SSH" situations.

---

## Additional Touch-Friendly Tweaks

```bash
sudo raspi-config
```

Consider enabling:
- Disable screen blanking (keeps display always on)
- Increase UI scaling if using a small screen (5" or 7")

---

## Why Onboard?

Other on-screen keyboards (Matchbox-keyboard, Florence, etc.) have limitations:
- No floating toggle button
- Unpredictable auto-popup behavior
- Poor touch ergonomics

Onboard is the only option that provides the "assistive floating button" UX similar to iOS.

---

## Summary Checklist

- [x] GUI auto-starts on boot
- [x] Touchscreen replaces mouse and keyboard
- [x] Floating translucent shortcut button
- [x] Keyboard hidden unless needed
- [x] Safe after power cycles
- [x] No manual commands required

---

## Optional: Display Calibration

If you need help with touch calibration or display orientation, note your:
- Screen size (5", 7", etc.)
- Orientation (portrait or landscape)
- Touchscreen brand/model

These details allow for precise calibration configuration.
