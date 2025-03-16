---
{"dg-publish":true,"permalink":"/projects/home-assistent/server/setup/05-docker/"}
---

# Docker Compose Architektur mit DeepSeek R1 LLM Integration

## 1. Architektur-Konzept

### Zentrale Komponenten

- DeepSeek R1 LLM
- Model Serving
- Inference-Optimierung
- Home Assistant Integration

## 2. Docker Compose Konfiguration

```yaml
version: '3.8'

# Netzwerk-Definition
networks:
  ai_network:
    driver: bridge
  home_network:
    driver: bridge

# Gemeinsame Umgebungsvariablen
x-common-variables: &common-variables
  TZ: Europe/Zurich
  CUDA_VISIBLE_DEVICES: 0  # Primäre GPU

# Volume-Definitionen
volumes:
  deepseek_models:
  inference_cache:
  model_storage:

services:
  #######################################
  # DEEPSEEK R1 LLM SERVICE
  #######################################
  
  # DeepSeek R1 Model Server
  deepseek-r1-server:
    build:
      context: ./deepseek-r1
      dockerfile: Dockerfile.server
    container_name: deepseek-r1-server
    volumes:
      - deepseek_models:/models
      - inference_cache:/cache
      - ./config:/app/config
    environment:
      <<: *common-variables
      MODEL_NAME: deepseek-r1
      INFERENCE_BACKEND: vllm  # Hochperformante Inference
      MAX_MODEL_LEN: 4096
      GPU_MEMORY_UTILIZATION: 0.9
    ports:
      - "8000:8000"  # Model Serving API
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    networks:
      - ai_network
    restart: unless-stopped

  # Inference Optimization Proxy
  inference-optimizer:
    build:
      context: ./inference-optimizer
      dockerfile: Dockerfile
    container_name: inference-optimizer
    volumes:
      - ./optimizer-config:/app/config
    environment:
      <<: *common-variables
      UPSTREAM_SERVER: http://deepseek-r1-server:8000
    ports:
      - "8001:8001"
    networks:
      - ai_network
    depends_on:
      - deepseek-r1-server
    restart: unless-stopped

  #######################################
  # HOME ASSISTANT INTEGRATION
  #######################################
  
  # AI Assistant Bridge
  ai-assistant-bridge:
    build:
      context: ./ai-assistant-bridge
      dockerfile: Dockerfile
    container_name: ai-assistant-bridge
    volumes:
      - ./bridge-config:/app/config
    environment:
      <<: *common-variables
      LLM_SERVER_URL: http://inference-optimizer:8001
      HOME_ASSISTANT_TOKEN: ${HOME_ASSISTANT_TOKEN}
    networks:
      - home_network
      - ai_network
    restart: unless-stopped

  #######################################
  # MODEL MANAGEMENT
  #######################################
  
  # Model Download & Management Service
  model-manager:
    build:
      context: ./model-manager
      dockerfile: Dockerfile
    container_name: model-manager
    volumes:
      - model_storage:/models
      - ./download-config:/app/config
    environment:
      <<: *common-variables
      HUGGINGFACE_TOKEN: ${HUGGINGFACE_TOKEN}
    networks:
      - ai_network
    restart: unless-stopped

  #######################################
  # MONITORING & OBSERVABILITY
  #######################################
  
  # Prometheus Monitoring
  model-metrics-exporter:
    build:
      context: ./metrics-exporter
      dockerfile: Dockerfile
    container_name: model-metrics-exporter
    environment:
      <<: *common-variables
      MONITOR_ENDPOINTS: >
        http://deepseek-r1-server:8000/metrics
        http://inference-optimizer:8001/metrics
    ports:
      - "9090:9090"
    networks:
      - ai_network
    restart: unless-stopped
```

## 3. Beispiel-Dockerfile für DeepSeek R1 Server

```dockerfile
# Dockerfile für DeepSeek R1 Model Server
FROM nvidia/cuda:12.1.0-base-ubuntu22.04

# Basis-Abhängigkeiten
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    git \
    wget

# Python-Abhängigkeiten
RUN pip3 install \
    torch \
    transformers \
    vllm \
    fastapi \
    uvicorn \
    huggingface_hub

# DeepSeek R1 Modell herunterladen
RUN python3 -c "from huggingface_hub import snapshot_download; \
    snapshot_download(repo_id='deepseek-ai/deepseek-r1', \
    token='${HUGGINGFACE_TOKEN}', \
    local_dir='/models/deepseek-r1')"

# Server-Anwendung
COPY server.py /app/server.py

# Start-Befehl
CMD ["uvicorn", "server:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 4. Beispiel-Server-Implementierung

```python
from fastapi import FastAPI
from vllm import LLM, SamplingParams
import torch

app = FastAPI()
model = LLM("/models/deepseek-r1", tensor_parallel_size=1)

@app.post("/generate")
async def generate(
    prompt: str, 
    max_tokens: int = 256, 
    temperature: float = 0.7
):
    sampling_params = SamplingParams(
        temperature=temperature,
        max_tokens=max_tokens
    )
    
    outputs = model.generate(prompt, sampling_params)
    return {"response": outputs[0].outputs[0].text}
```

## 5. Home Assistant Integration

```yaml
# configuration.yaml
conversation:
  intents:
    ask_ai:
      action:
        service: rest_command.query_deepseek_r1
        data:
          prompt: "{{text}}"

rest_command:
  query_deepseek_r1:
    url: 'http://ai-assistant-bridge:8002/query'
    method: POST
    payload: '{"prompt": "{{prompt}}"}'
```

## 6. Deployment-Strategie

### Installations-Skript

```bash
#!/bin/bash

# Erstelle notwendige Verzeichnisse
mkdir -p \
  ./deepseek-r1 \
  ./inference-optimizer \
  ./ai-assistant-bridge \
  ./model-manager \
  ./metrics-exporter

# Login für private Modelle
docker login huggingface.co

# Modelle herunterladen
docker-compose run --rm model-manager

# Services starten
docker-compose up -d
```

---

Diese Architektur bietet:

- Hochperformante DeepSeek R1 Integration
- Flexibles Model Serving
- Home Assistant Anbindung
- Monitoring und Metriken