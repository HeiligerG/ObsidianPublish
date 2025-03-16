---
{"dg-publish":true,"permalink":"/projects/home-assistent/server/setup/03-mqtt/"}
---

# MQTT Broker (Mosquitto) - Detaillierte Implementierungsplanung

## 1. Architektur-Überblick

### MQTT-Rolle im Smart Home

- Zentraler Kommunikationsbus
- Leichtgewichtige Nachrichtenübertragung
- Verbindung verschiedener IoT-Geräte und Dienste

### Systemanforderungen

- Minimale Ressourcen
- Hohe Verfügbarkeit
- Sichere Konfiguration
- Skalierbarkeit

## 2. Infrastruktur-Konfiguration

### Docker Compose Setup

```yaml
version: '3.8'

services:
  mosquitto:
    container_name: mosquitto
    image: eclipse-mosquitto:latest
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
      - ./mosquitto/certs:/mosquitto/certs
    ports:
      - "1883:1883"   # Standard MQTT Port
      - "8883:8883"   # MQTT über SSL
      - "9001:9001"   # WebSocket Port
    restart: unless-stopped
    networks:
      - home_network

networks:
  home_network:
    driver: bridge
```

## 3. Konfigurationsdateien

### Mosquitto Konfiguration (`mosquitto.conf`)

```conf
# Basis-Konfiguration
persistence true
persistence_location /mosquitto/data/
log_dest file /mosquitto/log/mosquitto.log

# Sicherheitseinstellungen
allow_anonymous false
password_file /mosquitto/config/passwd
acl_file /mosquitto/config/acl

# SSL/TLS Konfiguration
listener 8883
cafile /mosquitto/certs/ca.crt
certfile /mosquitto/certs/server.crt
keyfile /mosquitto/certs/server.key

# WebSocket-Listener
listener 9001
protocol websockets

# Verbindungslimits
max_connections 1024
max_queued_messages 1000
message_size_limit 10M
```

### Benutzer-Authentifizierung (`passwd`)

```bash
# Generieren von Benutzern
mosquitto_passwd -c /mosquitto/config/passwd homeassistant
mosquitto_passwd /mosquitto/config/passwd sensors
```

### Zugriffskontrollen (`acl`)

```conf
# ACL-Konfiguration
user homeassistant
topic readwrite #

user sensors
topic read sensors/#

# Globale Regeln
pattern read $SYS/#
```

## 4. Sicherheitsaspekte

### SSL/TLS-Zertifikate

```bash
# Selbstsignierte Zertifikate generieren
openssl req -new -x509 -days 365 -extensions v3_ca -keyout ca.key -out ca.crt
openssl req -new -key server.key -out server.csr
openssl x509 -req -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
```

### Firewall-Konfiguration

```bash
# UFW-Regeln
ufw allow 1883/tcp
ufw allow 8883/tcp
ufw allow 9001/tcp
```

## 5. Home Assistant Integration

### `configuration.yaml`

```yaml
mqtt:
  broker: mosquitto
  port: 1883
  username: !secret mqtt_username
  password: !secret mqtt_password
  discovery: true
  discovery_prefix: homeassistant
```

## 6. Performance-Optimierung

### Mosquitto-Konfiguration

```conf
# Performance-Tuning
connection_messages false
log_type error
log_timestamp true

# Keepalive und Reconnect
persistent_client_expiration 1m
reconnect_delay 5
reconnect_delay_max 30
```

## 7. Monitoring und Logging

### Prometheus Exporter

```yaml
services:
  mqtt-exporter:
    image: sapcc/mosquitto-exporter
    ports:
      - "9234:9234"
    command: 
      - '-mosquitto.server=mosquitto:1883'
      - '-mosquitto.username=exporter'
      - '-mosquitto.password=secret'
```

## 8. Backup-Strategie

### Backup-Skript

```bash
#!/bin/bash
BACKUP_DIR="/mnt/backup/mqtt"
mkdir -p $BACKUP_DIR

# Backup Konfigurationen
cp /mnt/data/mosquitto/config/* $BACKUP_DIR/
cp /mnt/data/mosquitto/data/* $BACKUP_DIR/

# Komprimierung
tar -czvf $BACKUP_DIR/mqtt-backup-$(date +%Y%m%d).tar.gz $BACKUP_DIR
```

## 9. Erweiterte Konfigurationen

### Bridge-Konfiguration (Optional)

```conf
# MQTT Bridge zu externen Brokern
connection external-bridge
address external.broker.com:1883
topic sensors/# out
topic commands/# in
remote_username bridgeuser
remote_password bridgepassword
```

## 10. Deployment-Skript

```bash
#!/bin/bash
# MQTT-Broker Deployment

# Verzeichnisse erstellen
mkdir -p ./mosquitto/{config,data,log,certs}

# Konfigurationsdateien kopieren
cp mosquitto.conf ./mosquitto/config/
cp passwd ./mosquitto/config/
cp acl ./mosquitto/config/

# Zertifikate generieren
./generate-certs.sh

# Docker Compose starten
docker-compose up -d mosquitto
```

---

Dies ist eine umfassende Planung für den MQTT-Broker.