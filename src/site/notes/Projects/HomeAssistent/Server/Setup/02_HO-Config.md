---
{"dg-publish":true,"permalink":"/projects/home-assistent/server/setup/02-ho-config/"}
---

# Home Assistant Core Configuration

## 1. Grundlegende Konfigurationsstruktur

### Verzeichnisstruktur

```
/mnt/data/homeassistant/
├── configuration.yaml
├── secrets.yaml
├── automations.yaml
├── scenes.yaml
├── scripts.yaml
├── integrations/
│   ├── mqtt.yaml
│   ├── wyoming.yaml
│   └── ai_services.yaml
└── custom_components/
```

## 2. Basis-Konfiguration (`configuration.yaml`)

```yaml
# Grundlegende Konfiguration
homeassistant:
  name: Mein Smart Home
  latitude: 47.3769  # Beispiel-Koordinaten
  longitude: 8.5417
  elevation: 400
  unit_system: metric
  time_zone: Europe/Zurich
  language: de
  external_url: https://home.beispiel.ch
  internal_url: http://homeassistant.local:8123

# Standardkomponenten
default_config:

# Logging
logger:
  default: warning
  logs:
    homeassistant.components.mqtt: info
    wyoming: debug

# Benutzer-Authentifizierung
person:
  - name: Hauptbenutzer
    id: hauptbenutzer
    user_id: hauptbenutzer_user_id

# Geografische Zone
zone:
  - name: Home
    latitude: 47.3769
    longitude: 8.5417
    radius: 100
    icon: mdi:home

# Netzwerk-Konfiguration
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 127.0.0.1
    - ::1
    - 192.168.1.0/24
```

## 3. Secrets Management (`secrets.yaml`)

```yaml
# Sensible Zugangsdaten und Konfigurationen
mqtt_username: !secret mqtt_username
mqtt_password: !secret mqtt_password
external_url: !secret external_url
internal_url: !secret internal_url
```

## 4. Automatisierungs-Grundgerüst (`automations.yaml`)

```yaml
- alias: "Startup Notification"
  trigger:
    - platform: homeassistant
      event: start
  action:
    - service: notify.persistent_notification
      data:
        message: "Home Assistant wurde gestartet"
        title: "System Status"

- alias: "Daily System Health Check"
  trigger:
    - platform: time
      at: "03:00:00"
  action:
    - service: system_health.info
```

## 5. AI-Dienste Integration (`integrations/ai_services.yaml`)

```yaml
# Wyoming-Protokoll für Sprachdienste
wyoming:
  # Speech-to-Text Konfiguration
  stt:
    - name: Whisper STT
      uri: tcp://whisper-stt:10300
  
  # Text-to-Speech Konfiguration
  tts:
    - name: CSM TTS
      uri: tcp://csm-tts:10600
  
  # Wake Word Erkennung
  wake:
    - name: Open Wake Word
      uri: tcp://openwakeword:10400

# Ollama LLM Integration
ollama:
  host: ollama
  port: 11434
  models:
    - llama3:8b
    - mistral:7b
```

## 6. Docker-Compose Integration

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: homeassistant/home-assistant:stable
    volumes:
      - /mnt/data/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Europe/Zurich
    network_mode: host
    devices:
      - /dev/ttyACM0  # Optional: Für ZigBee/Z-Wave Stick
    restart: unless-stopped
    privileged: true
```

## 7. Performance-Optimierungen

### Caching und Ressourcenmanagement

```yaml
# In configuration.yaml
recorder:
  purge_keep_days: 7
  commit_interval: 60
  db_url: postgresql://homeassistant:password@postgres/homeassistant

# Ressourcen-Limits im Docker-Compose
homeassistant:
  deploy:
    resources:
      limits:
        cpus: '4'
        memory: 2G
      reservations:
        cpus: '2'
        memory: 1G
```

## 8. Sicherheitsaspekte

### Zwei-Faktor-Authentifizierung

```yaml
# configuration.yaml
# Zwei-Faktor-Authentifizierung aktivieren
# Benötigt zusätzliche Konfiguration und Komponenten
```

## 9. Erweiterte Konfigurationsoptionen

### Backup-Strategie

```yaml
# Automatisches Backup (externes Skript empfohlen)
automation:
  - alias: "Tägliches Home Assistant Backup"
    trigger:
      - platform: time
        at: "02:00:00"
    action:
      - service: backup.create
        data:
          name: "Tägliches Backup {{ site.time | date: "%Y-%m-%d" }}"
```

---

Dies ist eine umfassende Grundkonfiguration für Home Assistant.
