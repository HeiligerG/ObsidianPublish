---
{"dg-publish":true,"permalink":"/projects/home-assistent/server/setup/07-ho-integ/"}
---

# Home Assistant DeepSeek R1 Integration

## 1. Integrationsarchitektur

### Komponenten

- DeepSeek R1 Model Server
- AI Assistant Bridge
- Home Assistant Komponenten
- REST-Schnittstellen
- Konversations-Handling

## 2. AI Assistant Bridge (`ai_assistant_bridge.py`)

```python
import os
import logging
from fastapi import FastAPI, Request, BackgroundTasks
from pydantic import BaseModel
import httpx
import json

# Logging-Konfiguration
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("ai-assistant-bridge")

class AssistantRequest(BaseModel):
    text: str
    context: dict = {}
    user_id: str = "default"

class AssistantResponse(BaseModel):
    response: str
    context_update: dict = {}

class DeepSeekIntegration:
    def __init__(self):
        self.model_url = os.getenv('DEEPSEEK_MODEL_URL', 'http://deepseek-r1-server:8000/generate')
        self.home_assistant_token = os.getenv('HOME_ASSISTANT_TOKEN')
        self.home_assistant_url = os.getenv('HOME_ASSISTANT_URL', 'http://homeassistant:8123')
        
        # Kontextmanagement
        self.user_contexts = {}
        
    async def generate_response(self, request: AssistantRequest) -> AssistantResponse:
        try:
            # Kontextaufbereitung
            context_prompt = self._prepare_context_prompt(request)
            
            # API-Anfrage an DeepSeek
            async with httpx.AsyncClient() as client:
                response = await client.post(
                    self.model_url, 
                    json={
                        "prompt": context_prompt,
                        "max_tokens": 256,
                        "temperature": 0.7
                    }
                )
                
                if response.status_code != 200:
                    raise Exception(f"Model Server Error: {response.text}")
                
                result = response.json()
                
                # Kontext aktualisieren
                new_context = self._update_context(
                    request.user_id, 
                    request.text, 
                    result['response']
                )
                
                return AssistantResponse(
                    response=result['response'],
                    context_update=new_context
                )
        
        except Exception as e:
            logger.error(f"Generation error: {e}")
            return AssistantResponse(response="Entschuldigung, es ist ein Fehler aufgetreten.")
    
    def _prepare_context_prompt(self, request: AssistantRequest) -> str:
        # Vorhandenen Kontext abrufen
        prev_context = self.user_contexts.get(request.user_id, {})
        
        # Kontextaufbereitung
        context_prompt = f"""
        Aktueller Kontext:
        {json.dumps(prev_context)}
        
        Neue Benutzereingabe: {request.text}
        
        Generiere eine hilfreiche und kontextbezogene Antwort:
        """
        
        return context_prompt
    
    def _update_context(self, user_id: str, user_input: str, model_response: str) -> dict:
        # Kontext-Update-Logik
        if user_id not in self.user_contexts:
            self.user_contexts[user_id] = {}
        
        self.user_contexts[user_id].update({
            "last_input": user_input,
            "last_response": model_response,
            "conversation_length": len(self.user_contexts[user_id].get('conversation_history', [])) + 1
        })
        
        return self.user_contexts[user_id]

# FastAPI App
app = FastAPI(title="AI Assistant Bridge")
deepseek_integration = DeepSeekIntegration()

@app.post("/query")
async def ai_query(request: AssistantRequest, background_tasks: BackgroundTasks):
    response = await deepseek_integration.generate_response(request)
    
    # Optionale Hintergrundaufgaben 
    # z.B. Logging, Benachrichtigungen
    background_tasks.add_task(
        log_conversation, 
        request.user_id, 
        request.text, 
        response.response
    )
    
    return response

def log_conversation(user_id: str, user_input: str, ai_response: str):
    # Logging-Mechanismus
    logger.info(f"User {user_id}: {user_input}")
    logger.info(f"AI Response: {ai_response}")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8002)
```

## 3. Home Assistant Konfiguration

### `configuration.yaml`

```yaml
# AI Assistant Konfiguration
conversation:
  intents:
    ai_assistant:
      action:
        service: rest_command.query_ai_assistant
        data:
          text: "{{text}}"
          user_id: "{{user.id}}"

# REST-Befehl für AI-Abfragen
rest_command:
  query_ai_assistant:
    url: 'http://ai-assistant-bridge:8002/query'
    method: POST
    content_type: 'application/json'
    payload: >
      {
        "text": "{{text}}",
        "user_id": "{{user.id}}"
      }

# Benachrichtigungen
notify:
  - platform: persistent_notification
    name: ai_assistant_notifications

# Sensible Konfigurationen in secrets.yaml
```

## 4. Automatisierungsbeispiele

```yaml
# Automatisierung: AI-Assistent Benachrichtigungen
automation:
  - alias: "AI Assistant Notification"
    trigger:
      platform: webhook
      webhook_id: ai_assistant_
```