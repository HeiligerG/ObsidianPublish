---
{"dg-publish":true,"permalink":"/projects/ai/konvertierung/"}
---

Dies ist eine detaillierte Anleitung zur Konvertierung von SafeTensor-Modellen in GGUF/quantisierte Modelme, inklusive Beispiele, Quellen und visuellen Darstellungen:

---

### **1. Grundlagen & Tools**
- **SafeTensor**: Ein sicheres Dateiformat für ML-Modelle (Hugging Face).
- **GGUF**: Das Nachfolgeformat von GGML, optimiert für Llama.cpp (GPU/CPU).
- **Quantisierung**: Reduziert die Modellgröße durch Präzisionsverkleinerung (z.B. FP32 → 4-Bit-Integer).

**Benötigte Tools**:
- `transformers` (Hugging Face)
- `llama.cpp` ([GitHub Repo](https://github.com/ggerganov/llama.cpp))
- Python/PyTorch

---

### **2. Konvertierungsprozess (Schritt-für-Schritt)**

#### **A. SafeTensor → PyTorch → GGUF**
```bash
# Schritt 1: SafeTensor mit Hugging Face laden
from transformers import AutoModelForCausalLM

model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-2-7b-chat-hf",
    use_safetensors=True
)

# Schritt 2: Konvertierung zu GGUF mit llama.cpp
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
python3 convert-hf-to-gguf.py \
  --model /pfad/zum/safetensor-modell \
  --outfile ./model/llama-2-7b-chat.gguf
```

#### **B. Quantisierung des GGUF-Modells**
```bash
# Kompilieren Sie llama.cpp (erfordert 'make' und C++-Compiler)
make

# Quantisierung (Beispiel: Q4_K_M)
./quantize ./model/llama-2-7b-chat.gguf ./model/llama-2-7b-chat-Q4_K_M.gguf Q4_K_M
```

---

### **3. Quantisierungsmethoden (Beispiele)**
| Typ       | Bits | Speicherbedarf | Genauigkeit |
|-----------|------|----------------|-------------|
| Q4_0      | 4    | Sehr klein     | Niedrig     |
| Q4_K_M    | 4    | Klein          | Mittel      |
| Q8_0      | 8    | Mittel         | Hoch        |

---

### **4. Visuelle Darstellung**
```
[SafeTensor (.safetensors)]
         |
         v
[PyTorch-Modell (.bin/.pth)]
         |
         v
[GGUF-Modell (.gguf)] 
         |
         v
[Quantisiertes GGUF (z.B. Q4_K_M)]
```

---

### **5. Beispiel: LLaMA-2-7B**
1. **Download** des SafeTensor-Modells von Hugging Face:
   ```python
   from huggingface_hub import snapshot_download
   snapshot_download(repo_id="meta-llama/Llama-2-7b-chat-hf")
   ```
2. **Konvertierung** mit `convert-hf-to-gguf.py` (Architektur angeben!):
   ```bash
   python3 convert-hf-to-gguf.py \
     --model ./Llama-2-7b-chat-hf \
     --outtype f16 \  # FP16-Konvertierung
     --outfile llama-2-7b-chat.gguf
   ```
3. **Quantisierung** auf 4-Bit:
   ```bash
   ./quantize llama-2-7b-chat.gguf llama-2-7b-chat-Q4_K_M.gguf Q4_K_M
   ```

---

### **6. Quellen & Referenzen**
- **llama.cpp-Dokumentation**: [GitHub Wiki](https://github.com/ggerganov/llama.cpp/wiki)
- **Hugging Face Safetensors**: [Offizielle Docs](https://huggingface.co/docs/safetensors/index)
- **Quantisierungshandbuch**: [LLM Performance Guide](https://github.com/ggerganov/llama.cpp/discussions/344)

---

### **7. Fehlerbehebung**
- **Fehlende Architektur**: Passen Sie `convert-hf-to-gguf.py` an (z.B. `--arch llama`).
- **RAM-Engpässe**: Nutzen Sie `--split` bei großen Modellen.
- **GPU-Unterstützung**: Kompilieren Sie llama.cpp mit CUDA (`make LLAMA_CUDA=1`).

---

Diese Anleitung deckt den vollständigen Workflow von SafeTensor zu einem quantisierten GGUF-Modell ab. Für spezifische Modelle (z.B. Mistral, Phi-3) können zusätzliche Parameter erforderlich sein.