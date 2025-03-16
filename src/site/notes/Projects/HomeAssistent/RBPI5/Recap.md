---
{"dg-publish":true,"permalink":"/projects/home-assistent/rbpi-5/recap/"}
---

# Was gut funktioniert hat und Beispiele für künftige Setups

## Erfolgreiche Komponenten deines bisherigen Systems

Das bisherige Setup auf dem Raspberry Pi hatte trotz der Herausforderungen mit CSM bereits einige gut funktionierende Elemente:

1. **Home Assistant mit LocalAI und TinyLlama**: Diese Kombination ermöglichte erste KI-Funktionen für Textverständnis und -generierung
2. **Piper für Sprachausgabe**: Piper lief stabil und bot qualitativ bessere Sprachausgabe als espeak-ng
3. **Vosk für Spracherkennung**: Die Container für Deutsch und Englisch funktionierten zuverlässig
4. **Wyoming-Protokoll**: Das Protokoll zur Standardisierung der Kommunikation zwischen Sprachkomponenten
5. **Docker-Container-Struktur**: Die Grundstruktur mit separaten Containern hat sich bewährt

## Beispiel für ein optimiertes Setup auf deinem neuen System

Hier ist ein Beispiel, wie ein vollständig funktionierendes System auf dem leistungsstarken Server aussehen könnte:

```yaml
# docker-compose.yml für optimiertes Setup
version: '3'

services:
  # 1. Home Assistant Core
  homeassistant:
    container_name: homeassistant
    image: homeassistant/home-assistant:stable
    volumes:
      - ./config:/config
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "8123:8123"
    restart: unless-stopped
    network_mode: host
    privileged: true
    depends_on:
      - mosquitto
      - postgres

  # 2. Text-Verstehen mit Llama 3
  localai:
    container_name: localai
    image: quay.io/go-skynet/local-ai:latest
    ports:
      - "8080:8080"
    environment:
      - DEBUG=true
      - MODELS_PATH=/models
      - CONTEXT_SIZE=4096
      - GO_MEMORY_LIMIT=8192MiB 
      - THREADS=8
      - CUDA_VISIBLE_DEVICES=0
    volumes:
      - ./localai/models:/models
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
  
  # 3. Spracherkennung mit OpenAI Whisper
  wyoming-whisper:
    container_name: wyoming-whisper 
    image: rhasspy/wyoming-whisper
    restart: unless-stopped
    volumes:
      - ./whisper-data:/data
    command: >-
      --model medium
      --language de,en
      --uri tcp://0.0.0.0:10300
      --device cuda
    ports:
      - "10300:10300"
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [compute]

  # 4. Hochqualitative Sprachausgabe mit CSM
  csm-server:
    container_name: csm-server
    build: ./csm-server
    restart: unless-stopped
    ports:
      - "8765:8765"
    volumes:
      - ./csm-models:/app/models
    environment:
      - NVIDIA_VISIBLE_DEVICES=0
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu]

  # 5. Wyoming-Integration für CSM
  wyoming-csm:
    container_name: wyoming-csm
    build: ./wyoming-csm
    restart: unless-stopped
    command: >-
      --csm-url http://csm-server:8765
      --uri tcp://0.0.0.0:10600
    ports:
      - "10600:10600"
    depends_on:
      - csm-server

  # 6. Wakeword-Erkennung mit OpenWakeWord
  wyoming-openwakeword:
    container_name: wyoming-openwakeword
    image: rhasspy/wyoming-openwakeword
    restart: unless-stopped
    volumes:
      - ./openwakeword-data:/data
    command: >-
      --model hey_sira,computer
      --uri tcp://0.0.0.0:10400
    ports:
      - "10400:10400"

  # 7. Wyoming-Satellite für Gerätekommunikation
  wyoming-satellite:
    container_name: wyoming-satellite
    image: rhasspy/wyoming-satellite
    restart: unless-stopped
    devices:
      - /dev/snd:/dev/snd
    command: >-
      --mic-command "arecord -D hw:CARD=Jabra,DEV=0 -r 16000 -c 1 -f S16_LE -t raw"
      --snd-command "aplay -D hw:CARD=Jabra,DEV=0 -r 22050 -c 2 -f S16_LE -t raw"
      --stt-uri tcp://wyoming-whisper:10300
      --tts-uri tcp://wyoming-csm:10600
      --wake-uri tcp://wyoming-openwakeword:10400
      --uri tcp://0.0.0.0:10500
    ports:
      - "10500:10500"
    depends_on:
      - wyoming-whisper
      - wyoming-csm
      - wyoming-openwakeword

  # 8. Zusätzliche Services
  mosquitto:
    container_name: mosquitto
    image: eclipse-mosquitto:latest
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
    restart: unless-stopped

  postgres:
    container_name: postgres
    image: postgres:14
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_PASSWORD=homeassistant
      - POSTGRES_USER=homeassistant
      - POSTGRES_DB=homeassistant
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    restart: unless-stopped
```

## CSM-Server Implementierung

Für den CSM-Server könntest du ein einfaches Flask-basiertes Interface zum CSM-Modell erstellen:

```python
# csm-server/app.py
from flask import Flask, request, send_file
import torch
import torchaudio
from generator import load_csm_1b
import tempfile
import os
import logging

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)

# Globale Variablen
model = None
device = "cuda" if torch.cuda.is_available() else "cpu"

@app.route('/health', methods=['GET'])
def health_check():
    return {"status": "ok", "device": device}

@app.route('/generate', methods=['POST'])
def generate_speech():
    global model
    
    if model is None:
        # Lade das Modell bei Bedarf
        model_path = "/app/models/csm-1b.pt"
        model = load_csm_1b(model_path, device)
    
    data = request.json
    text = data.get('text', '')
    speaker = data.get('speaker', 1)  # 1 für weibliche Stimme
    
    # Generiere Audio
    audio = model.generate(
        text=text,
        speaker=speaker,
        context=[],
        max_audio_length_ms=10000,
    )
    
    # Speichere Audio temporär
    temp_file = tempfile.NamedTemporaryFile(delete=False, suffix='.wav')
    temp_file.close()
    
    torchaudio.save(temp_file.name, audio.unsqueeze(0).cpu(), model.sample_rate)
    
    # Sende die Datei zurück
    response = send_file(temp_file.name, mimetype='audio/wav')
    
    # Lösche die Datei nach dem Senden
    os.unlink(temp_file.name)
    
    return response

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8765)
```

## Wyoming-CSM Adapter

Der Wyoming-CSM Adapter würde den CSM-Server mit dem Wyoming-Protokoll verbinden:

```python
# wyoming-csm/bridge.py
import asyncio
import logging
import argparse
import io
import wave
import requests
from wyoming.info import Info, Describe, TtsModel, TtsVoice
from wyoming.server import AsyncServer
from wyoming.tts import Synthesize, SynthesisComplete
from wyoming.transport import AsyncMessageReader, AsyncMessageWriter

_LOGGER = logging.getLogger(__name__)

async def tts_handle_conn(
    reader: AsyncMessageReader,
    writer: AsyncMessageWriter,
    csm_url: str,
):
    try:
        # Sende Info-Nachricht
        info = Info(
            tts=[
                TtsModel(
                    model_id="csm",
                    names=["CSM"],
                    description="Sesame CSM Text-to-Speech",
                    attribution="Sesame Labs",
                    languages=["de", "en"],
                    voices=[
                        TtsVoice(
                            voice_id="female",
                            name="Female",
                            description="Weibliche Stimme",
                        ),
                        TtsVoice(
                            voice_id="male",
                            name="Male",
                            description="Männliche Stimme",
                        ),
                    ],
                )
            ]
        )
        await writer.write(info.event())

        # Verarbeitungsschleife
        async for msg in reader:
            if Describe.is_type(msg.type):
                await writer.write(info.event())
            elif Synthesize.is_type(msg.type):
                synthesize = Synthesize.from_event(msg)
                text = synthesize.text
                
                # Wähle Stimme
                speaker = 1  # Weiblich als Standard
                for voice in synthesize.voice:
                    if voice.voice_id == "male":
                        speaker = 0
                        break
                
                _LOGGER.info(f"Synthesizing: '{text}' (speaker={speaker})")
                
                # Sende Anfrage an CSM-Server
                try:
                    response = requests.post(
                        f"{csm_url}/generate",
                        json={"text": text, "speaker": speaker},
                        timeout=30
                    )
                    
                    # Konvertiere Audio
                    with io.BytesIO(response.content) as wav_file:
                        with wave.open(wav_file, "rb") as wav:
                            audio_bytes = wav.readframes(wav.getnframes())
                            rate = wav.getframerate()
                            width = wav.getsampwidth()
                            channels = wav.getnchannels()
                    
                    # Sende fertige Synthese
                    synth_complete = SynthesisComplete(
                        audio_bytes=audio_bytes,
                        sample_rate=rate,
                        sample_width=width,
                        channels=channels,
                    )
                    await writer.write(synth_complete.event())
                except Exception as e:
                    _LOGGER.error(f"Error in TTS: {e}")
    except Exception:
        _LOGGER.exception("Unexpected error")
    finally:
        writer.close()

async def main():
    parser = argparse.ArgumentParser()
    parser.add_argument(
        "--csm-url", 
        default="http://csm-server:8765", 
        help="URL of CSM server"
    )
    parser.add_argument(
        "--uri", 
        default="tcp://0.0.0.0:10600", 
        help="URI to listen on"
    )
    parser.add_argument(
        "--debug", 
        action="store_true", 
        help="Log DEBUG messages"
    )
    args = parser.parse_args()
    
    logging.basicConfig(
        level=logging.DEBUG if args.debug else logging.INFO,
        format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    )
    
    server = AsyncServer.from_uri(args.uri)
    _LOGGER.info("Starting server at %s", server)
    
    await server.run(
        lambda r, w: tts_handle_conn(r, w, args.csm_url)
    )

if __name__ == "__main__":
    asyncio.run(main())
```

## Home Assistant Konfiguration

In deiner Home Assistant configuration.yaml würdest du folgendes ergänzen:

```yaml
# Wyoming integrieren
wyoming:
  satellite:
    - id: jabra_satellite
      host: localhost
      port: 10500
  stt:
    - id: whisper_stt
      host: localhost
      port: 10300
  tts:
    - id: csm_tts
      host: localhost
      port: 10600
  wake:
    - id: openwakeword
      host: localhost
      port: 10400

# Assist-Pipeline konfigurieren
assist_pipeline:
  language: de
  conversation_engine: local_ai
  tts_engine: wyoming
  tts_voice: csm/female
  stt_engine: wyoming
  stt_platform: wyoming
  wake_word_entity: wyoming.openwakeword
  wake_word: hey_sira

# LocalAI-Anbindung
local_ai:
  url: http://localhost:8080
  default_model: llama-3-8b
```