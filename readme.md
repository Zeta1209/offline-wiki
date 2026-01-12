# Projet Offline Wikipedia sur Raspberry Pi Zero W

## Objectif

Créer un point d'accès Wi-Fi autonome servant Wikipedia hors-ligne via Kiwix :

- **Wi-Fi intégré (wlan0)** → Point d'accès "OfflineWiki"
- **Dongle USB Wi-Fi (wlan1)** → Connexion au réseau maison (LAN/Internet)
- **Kiwix** → Serveur HTTP pour Wikipedia offline

---

## Matériel requis

| Composant | Notes |
|-----------|-------|
| Raspberry Pi Zero / Zero W | Avec Wi-Fi intégré |
| Carte microSD | 512 GB recommandée pour les fichiers ZIM |
| Dongle Wi-Fi USB | Pour la connexion LAN |
| Hub USB alimenté | Évite les problèmes d'alimentation |
| Câble alimentation | USB-C ou micro-USB selon modèle |
| Adaptateur Mini-HDMI | Pour configuration initiale (optionnel) |

---

## Étape 0 — Préparer la carte SD

1. Télécharger **Raspberry Pi OS Lite (32-bit)**
2. Flasher avec **Raspberry Pi Imager**
3. Activer SSH :
   ```bash
   touch /boot/ssh
   ```
4. Insérer la carte et démarrer le Pi

---

## Étape 1 — Connexion SSH initiale

```bash
# Trouver l'IP du Pi
hostname -I

# Se connecter
ssh piwiki@<IP_DU_PI>

# Changer le mot de passe par défaut
passwd
```

---

## Étape 2 — Préparer Kiwix et les fichiers ZIM

### Créer le dossier de stockage

```bash
sudo mkdir -p /data/zim
sudo chown -R piwiki:piwiki /data
```

### Transférer les fichiers ZIM (depuis ton PC)

Utiliser WinSCP

### Tester Kiwix manuellement

```bash
kiwix-serve --port=8080 /data/zim/*.zim
```

---

## Étape 3 — Service systemd pour Kiwix

### Créer le fichier service

```bash
sudo nano /etc/systemd/system/kiwix.service
```

**Contenu :**

```ini
[Unit]
Description=Kiwix Offline Wikipedia
After=network.target

[Service]
User=pi
ExecStart=/usr/bin/kiwix-serve --port=8080 /data/zim/*.zim
Restart=always

[Install]
WantedBy=multi-user.target
```

### Activer le service

```bash
sudo systemctl daemon-reload
sudo systemctl enable kiwix
sudo systemctl start kiwix
sudo systemctl status kiwix --no-pager
```

---

## Étape 4 — Configurer le Point d'Accès Wi-Fi

### 4.1 — Installer hostapd et dnsmasq

```bash
sudo apt update
sudo apt install hostapd dnsmasq
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
```

### 4.2 — IP fixe pour wlan0 (AP)

```bash
sudo nano /etc/dhcpcd.conf
```

**Ajouter à la fin :**

```
interface wlan0
    static ip_address=192.168.4.1/24
    nohook wpa_supplicant
```

### 4.3 — Configurer hostapd

```bash
sudo nano /etc/hostapd/hostapd.conf
```

**Contenu :**

```
interface=wlan0
driver=nl80211
ssid=OfflineWiki
hw_mode=g
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=MonMotDePasseWiFi
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

**Activer la config :**

```bash
sudo nano /etc/default/hostapd
```

**Modifier/ajouter :**

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### 4.4 — Configurer dnsmasq

```bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
```

**Contenu :**

```
interface=wlan0
dhcp-range=192.168.4.2,192.168.4.20,255.255.255.0,24h
```

### 4.5 — Désactiver wpa_supplicant sur wlan0

```bash
sudo systemctl disable wpa_supplicant
sudo systemctl stop wpa_supplicant
sudo ip link set wlan0 down
sudo ip link set wlan0 up
```

### 4.6 — Tester le point d'accès

```bash
sudo hostapd -dd /etc/hostapd/hostapd.conf
```

Vérifie que le SSID **OfflineWiki** apparaît sur ton appareil.

---

## Étape 5 — Configurer le dongle USB (client Wi-Fi)

### Identifier l'interface

```bash
iw dev
```

Le dongle sera probablement `wlan1`.

### Configurer la connexion au réseau maison

```bash
sudo nano /etc/wpa_supplicant/wpa_supplicant-wlan1.conf
```

**Contenu :**

```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=CA

network={
    ssid="MonReseauMaison"
    psk="MotDePasseMaison"
}
```

### Connecter le dongle

```bash
sudo wpa_supplicant -B -i wlan1 -c /etc/wpa_supplicant/wpa_supplicant-wlan1.conf
sudo dhclient wlan1
```

### Vérifier la connexion

```bash
ip addr show wlan1
ping 8.8.8.8
```

---

## Étape 6 — Activer tout au démarrage

```bash
sudo systemctl enable hostapd
sudo systemctl enable dnsmasq
sudo systemctl start hostapd
sudo systemctl start dnsmasq
```

---

## Résultat final

| Interface | Rôle | Réseau |
|-----------|------|--------|
| wlan0 | Point d'accès | OfflineWiki (192.168.4.1) |
| wlan1 | Client Wi-Fi | Réseau maison |

### Accès Wikipedia offline

Connecte-toi au Wi-Fi **OfflineWiki** puis ouvre :

```
http://192.168.4.1:8080
```

### Accès SSH depuis le LAN

```bash
ssh pi@<IP_WLAN1>
```

---

## Notes importantes

- **Le Pi Zero W ne peut pas faire AP + client sur la même interface** — le dongle USB est obligatoire pour les deux fonctions simultanées
- **Hub USB alimenté recommandé** — évite les problèmes de puissance avec le dongle
- **Ajouter des fichiers ZIM** — copie-les dans `/data/zim` et redémarre Kiwix :
  ```bash
  sudo systemctl restart kiwix
  ```

---

## Dépannage

### Le point d'accès n'apparaît pas

```bash
sudo systemctl status hostapd
journalctl -u hostapd -e
```

### Kiwix ne démarre pas

```bash
sudo systemctl status kiwix
ls -la /data/zim/
```

### Pas de connexion Internet via le dongle

```bash
ip addr show wlan1
ping -I wlan1 8.8.8.8
```
