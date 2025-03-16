---
{"dg-publish":true,"permalink":"/projects/home-assistent/server/setup/01-base/"}
---


```

# Mid-Tier Home Assistant & KI-Server - Implementierungsplanung

## 1. Systemarchitektur-Überblick

### Hardware-Spezifikationen

- **CPU**: AMD Ryzen 9 7950X (16 Kerne, 5.7 GHz)
- **RAM**: 64GB DDR5-6000MHz
- **GPU**: NVIDIA RTX 4070 12GB
- **Storage**:
    - 2TB NVMe OS-Disk (Samsung 990 Pro)
    - 2TB NVMe Daten-Disk (Kingston KC3000)
    - 8TB NAS-optimierte HDD (Seagate IronWolf)

### Primäre Einsatzbereiche

- Home Assistant
- Docker-Containerisierung
- KI-Modell-Hosting
- Medienserver
- Virtualisierung

## 2. Storage-Strategie

### Disk-Layout

```
/dev/nvme0n1 (Samsung 990 Pro 2TB):
├── /boot/efi       # 512MB
├── /               # 100GB Root
├── /home           # 200GB Benutzer
└── /var/lib/docker # 1TB Docker Volumes

/dev/nvme1n1 (Kingston KC3000 2TB):
├── /opt/ai-models  # 500GB KI-Modelle
├── /var/lib/vms    # 1TB Virtuelle Maschinen
└── /mnt/data       # Restlicher Speicher für Daten

/dev/sda (Seagate IronWolf 8TB):
└── /mnt/backup     # Backup-Speicher für wichtige Daten
```

### Backup-Strategie

- Nächtliche Backups der kritischen Systeme
- Backup-Rotation (7 täglich, 4 wöchentlich, 12 monatlich)
- Verschlüsselte Backups

## 3. Docker & Containermanagement

### Docker-Konfiguration

```yaml
# /etc/docker/daemon.json
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  }
}
```

### Docker Compose Basis-Struktur

```yaml
version: '3.8'

networks:
  home_network:
    driver: bridge

services:
  homeassistant:
    image: homeassistant/home-assistant:stable
    volumes:
      - /mnt/data/homeassistant:/config
    network_mode: host
    restart: unless-stopped

  nginx-proxy:
    image: jwilder/nginx-proxy
    volumes:
      - /var/run/docker.sock:/tmp/docker.sock
    ports:
      - "80:80"
      - "443:443"
```

## 4. KI & Machine Learning Setup

### GPU-Optimierung

- CUDA 12.x
- cuDNN 8.x
- TensorRT für Inferenz-Beschleunigung

### KI-Modell-Management

```bash
#!/bin/bash
# Modell-Download und Verwaltungsskript

# Verzeichnis für Modelle
mkdir -p /opt/ai-models/{llms,speech,vision}

# Beispiel: Llama 3 Modell herunterladen
ollama pull llama3:8b

# CSM Speech Modell
wget -P /opt/ai-models/speech https://huggingface.co/sesame/csm-1b/resolve/main/ckpt.pt
```

## 5. Netzwerk-Konfiguration

### Netzwerk-Design

- Internes Docker-Netzwerk
- Isolierte Container-Kommunikation
- Reverse Proxy für Dienste

### Firewall-Regeln (UFW)

```bash
# Basis-Firewall-Konfiguration
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow from 192.168.1.0/24  # Lokales Netzwerk
ufw enable
```

## 6. Monitoring & Wartung

### Monitoring-Stack

- Prometheus
- Grafana
- Node Exporter
- NVIDIA DCGM Exporter

### Beispiel Prometheus Konfiguration

```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
  
  - job_name: 'nvidia_gpu'
    static_configs:
      - targets: ['localhost:9445']
```

## 7. Optimierungen

### Systemkonfigurationen

```bash
# /etc/sysctl.conf Optimierungen
vm.swappiness = 10
vm.dirty_ratio = 10
vm.dirty_background_ratio = 5
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

### Performance-Tuning

- CPU-Gouverneur auf "performance"
- Thermische Verwaltung
- Energiesparoptimierungen deaktivieren

## 8. Sicherheitsaspekte

### Basis-Sicherheitsmaßnahmen

- Fail2Ban
- Regelmäßige Sicherheitsupdates
- Verschlüsselte Volumes
- Minimale Berechtigungen

### SSH-Härtung

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
AllowUsers homeserver
```

## 9. Erweiterbarkeit

### Upgrade-Pfade

- Zusätzliche NVMe-Slots
- Erweiterbare RAM-Slots
- PCIe 5.0 für zukünftige GPU-Upgrades

## 10. Deployment-Strategie

### Initialisierungsskript

```bash
#!/bin/bash
# Vollständiges System-Setup

# Systemupdate
sudo apt update && sudo apt upgrade -y

# Docker-Installation
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Nvidia Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

# Dienste starten
docker-compose up -d
```

---

Dies ist eine umfassende Planungsdokumentation für einen Mid-Tier Home Assistant & KI-Server.