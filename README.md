# OTS Internal Tooling: LM Studio — Windows Setup Guide

> Run local AI models on Windows with full privacy, zero API cost, and OpenAI-compatible API access.

---

## Table of Contents

1. [Download & Install LM Studio](#1-download--install-lm-studio)
2. [System Capacity Planning](#2-system-capacity-planning)
3. [Choosing the Right Model](#3-choosing-the-right-model)
4. [Download & Load a Model](#4-download--load-a-model)
5. [Integrate with VS Code](#5-integrate-with-vs-code)
6. [Use LM Studio from Code](#6-use-lm-studio-from-code)

---

## 1. Download & Install LM Studio

### Step 1 — Check System Requirements

**Minimum:**
- Windows 10 64-bit (build 1903+)
- 16 GB RAM
- 10 GB free disk space
- Intel/AMD CPU with AVX2 support
- Any GPU or CPU-only mode

**Recommended:**
- Windows 11 64-bit
- 32 GB+ RAM
- SSD with 50 GB+ space
- NVIDIA GPU with 8 GB+ VRAM
- CUDA 12.x drivers

> **⚠ AVX2 is required.** Confirm support by looking up your CPU on [Intel ARK](https://ark.intel.com) or AMD specs. Nearly all CPUs from 2013+ have it.

---

### Step 2 — Download the Installer

Go to **https://lmstudio.ai** and click **"Download for Windows"**, or use PowerShell:

```powershell
# Download with PowerShell (replace version as needed)
$url  = "https://releases.lmstudio.ai/windows/LM-Studio-0.3.x-Setup.exe"
$dest = "$env:USERPROFILE\Downloads\LM-Studio-Setup.exe"
Invoke-WebRequest -Uri $url -OutFile $dest
Start-Process $dest
```

> **Tip:** LM Studio also ships a **portable .zip** — useful for USB drives or machines without admin rights. Extract and run `lm-studio.exe` directly.

---

### Step 3 — Run the Installer

Double-click the downloaded `.exe`. If Windows SmartScreen appears, click **More info → Run anyway**.

- Accept the license agreement
- Choose install path (default: `C:\Users\You\AppData\Local\Programs\LM Studio`)
- Check "Create desktop shortcut" and "Start at login" (optional)
- Click **Install** — takes under 60 seconds
- Launch LM Studio when prompted

---

### Step 4 — Install the CLI (`lms`)

LM Studio ships with a powerful command-line interface. Enable it once from the app, then use it from any terminal.

From the LM Studio app: **Developer tab → "Enable CLI"**. This adds `lms` to your PATH.

```powershell
lms --version    # confirm install
lms status       # check server status
lms ls           # list downloaded models
```

---

### Step 5 — Configure GPU Acceleration

This is the single biggest performance lever. LM Studio auto-detects your GPU but you should verify the backend.

| GPU Vendor | Best Backend | Setting in LM Studio |
|---|---|---|
| NVIDIA (GTX 10xx+) | CUDA (llama.cpp) | Settings → GPU → CUDA |
| AMD (RX 6000+) | ROCm (experimental) | Settings → GPU → Vulkan |
| Intel Arc (A-series) | SYCL / Vulkan | Settings → GPU → Vulkan |
| No dedicated GPU | CPU-only (llama.cpp) | Settings → GPU → Disabled |

> **ℹ️ Info:** For NVIDIA users: open Settings → Hardware and set **GPU Layers** to **-1** (auto). LM Studio will offload as many layers to VRAM as possible, maximizing speed.

---

## 2. System Capacity Planning

Choosing a model that fits your hardware is critical. The wrong choice means slow generation, OOM crashes, or degraded quality from over-quantization.

### The VRAM Formula

```
VRAM_needed = (Params × Bits/8) + (ctx × 0.5 MB)
```

**Example:** 7B model at Q4_K_M (4 bits) + 4096 ctx = (7×10⁹ × 4/8) ÷ 10⁹ + 2 ≈ **5.5 GB VRAM**

### Quantization Reference

| Quant | Bits/param | Description |
|---|---|---|
| Q2 | ~2 bits | Smallest size, most quality loss |
| Q4 | ~4 bits | Sweet spot — recommended default |
| Q6 | ~6 bits | Near-lossless quality |
| F16 | 16 bits | Full quality, maximum VRAM usage |

---

### Hardware Tier Table

| Tier | VRAM | System RAM | Recommended Models | Expected Speed | Notes |
|---|---|---|---|---|---|
| Entry | 4 GB | 16 GB | 3B–7B Q2_K / Q3_K | 8–18 tok/s | Usable for chat, limited ctx |
| Mid | 8 GB | 32 GB | 7B Q4_K_M / 13B Q2 | 20–45 tok/s | Good balance, RTX 3070/4060 Ti |
| Pro | 16 GB | 64 GB | 13B Q6 / 34B Q4_K_M | 25–40 tok/s | RTX 3090/4080, excellent quality |
| Max | 24 GB+ | 128 GB | 70B Q4 / 34B Q8 / MoE | 10–25 tok/s | RTX 3090/4090, top-tier output |
| CPU-only | None | 32 GB+ | 7B Q4_K_M (RAM-based) | 2–8 tok/s | Slow but works; needs fast RAM |

---

### Context Length vs VRAM Cost

| Context | VRAM Impact |
|---|---|
| 2048 | Minimal |
| 4096 | +~0.5 GB |
| 8192 | +~2 GB |
| 32768 | +~8 GB |

> Don't set context higher than you actually need.

---

### GPU Layer Offloading

Layers offloaded to GPU run ~10× faster than CPU. Set GPU Layers to `-1` for auto-max.

| Config | Speed |
|---|---|
| 0 layers (CPU only) | 2–6 tok/s |
| ~50% layers on GPU | 12–20 tok/s |
| 100% layers on GPU | 25–55 tok/s |

If the model doesn't fully fit in VRAM, LM Studio automatically splits layers between VRAM and RAM.

---

### Optimal Settings Reference

```
# Optimal settings for a 7B model on 8 GB VRAM GPU

GPU Layers     : -1          # auto-max offload
Context Length : 4096        # balance ctx vs VRAM
Batch Size     : 512         # higher = faster prompt processing
Threads        : 8           # match your CPU physical core count
Flash Attn     : enabled     # reduces VRAM ~20%, speeds attention
mmap           : enabled     # memory-mapped I/O for fast model load
mlock          : disabled    # only enable if you have ample RAM

# Sampler settings for consistent, quality output
Temperature    : 0.7
Top-P          : 0.95
Repeat Penalty : 1.1
Min-P          : 0.05
```

---

## 3. Choosing the Right Model

### Coding Models

| Model | Size | VRAM | Best For |
|---|---|---|---|
| Qwen2.5-Coder-7B-Instruct (Q4_K_M) | ~4.5 GB | 6 GB | Code gen, completion, debugging — top pick for VS Code |
| DeepSeek-Coder-V2 16B MoE (Q4) | ~10 GB | 12 GB | Complex codebases, refactoring |

### General Models

| Model | Size | VRAM | Best For |
|---|---|---|---|
| Llama-3.2-8B-Instruct (Q4_K_M) | ~5.5 GB | 6 GB | Chat, writing, Q&A |
| Mistral-7B-Instruct-v0.3 (Q5_K_M) | ~5 GB | 6 GB | General tasks, summarization |

### Reasoning Models

| Model | Size | VRAM | Best For |
|---|---|---|---|
| Phi-4 14B (Q4_K_M) | ~9 GB | 10 GB | Math, logic, STEM |
| Gemma-3-27B-IT (Q4) | ~18 GB | 20 GB | Deep analysis, multilingual |

> **Quantization tip:** Q4_K_M is the best all-around choice — it loses less than 1% quality vs full F16 while using half the VRAM. Q5_K_M and Q6_K are great if you have VRAM headroom. Avoid Q2 unless severely constrained.

---

## 4. Download & Load a Model

### Method A — UI (LM Studio App)

1. Click the **🔍 Search** icon in the left sidebar to open model discovery
2. Type a model name (e.g., `qwen2.5-coder`) — results pull from Hugging Face
3. Expand the model card, choose a quant variant (look for `Q4_K_M` or `Q5_K_M`), click **Download**
4. Switch to the **Chat** tab, click the model selector, choose your model, and wait for the green "Model loaded" indicator

> LM Studio shows file size before download and estimates whether it fits in your VRAM — pay attention to the green/yellow/red indicators.

---

### Method B — CLI (`lms`)

```powershell
# Search for available models
lms search "qwen2.5-coder"

# Download a specific model (auto-picks best quant for your hardware)
lms get Qwen/Qwen2.5-Coder-7B-Instruct-GGUF

# Download a specific quantization
lms get Qwen/Qwen2.5-Coder-7B-Instruct-GGUF --quant q4_k_m

# List all downloaded models
lms ls

# Load a model into the server
lms load qwen2.5-coder-7b-instruct

# Load with custom context length
lms load qwen2.5-coder-7b-instruct --context-length 8192

# Unload to free VRAM
lms unload --all

# Quick chat from terminal
lms chat --model qwen2.5-coder-7b-instruct
```

---

### Method C — Local GGUF File

If you've downloaded a `.gguf` file manually from Hugging Face, place it in the models directory:

```
C:\Users\YourName\.cache\lm-studio\models\
  └── lmstudio-community\
      └── Qwen2.5-Coder-7B-Instruct-GGUF\
          └── qwen2.5-coder-7b-instruct-q4_k_m.gguf
```

- Go to **My Models** tab in LM Studio — the model appears automatically
- Or go to **Settings → Change models folder** to point to a custom directory

---

## 5. Integrate with VS Code

### Prerequisites — Start the Local Server

In LM Studio, click the **↔ Developer** tab → click **Start Server**.  
It binds to `http://localhost:1234`. Keep LM Studio running while you code.

---

### Option A — Continue.dev Extension

The best open-source AI coding assistant for VS Code. Supports inline completion, chat, and code edits powered by your local model.

**1. Install the extension:**

```
ext install Continue.continue
```

**2. Edit `~/.continue/config.json`:**

```json
{
  "models": [{
    "title": "LM Studio",
    "provider": "lmstudio",
    "model": "AUTODETECT"
  }],
  "tabAutocompleteModel": {
    "title": "Qwen2.5-Coder",
    "provider": "lmstudio",
    "model": "qwen2.5-coder-7b-instruct"
  }
}
```

> Continue has a built-in `lmstudio` provider — no URL config needed. It auto-discovers the running model.

---

### Option B — Cline (Agentic Assistant)

Cline is an agentic coding assistant that can read/write files, run terminal commands, and browse. Works with local models via the OpenAI-compatible API.

**1. Install Cline:**

```
ext install saoudrizwan.claude-dev
```

**2. Configure in Cline Settings:**

| Setting | Value |
|---|---|
| API Provider | OpenAI Compatible |
| Base URL | `http://localhost:1234/v1` |
| API Key | `lm-studio` (any value) |
| Model | `qwen2.5-coder-7b-instruct` |

---

### Any OpenAI-Compatible Extension

Almost any AI coding extension that supports a custom endpoint works with LM Studio:

```
Base URL  : http://localhost:1234/v1
API Key   : lm-studio   # ignored, can be any string
Model ID  : your-model-name   # match exactly what `lms ps` shows
```

**Test with curl (PowerShell):**

```powershell
curl http://localhost:1234/v1/chat/completions `
  -H "Content-Type: application/json" `
  -d '{"model":"qwen2.5-coder-7b-instruct","messages":[{"role":"user","content":"Hello!"}]}'
```

---

## 6. Use LM Studio from Code

The local server is fully OpenAI-compatible. Point your existing OpenAI client at `localhost:1234` — nothing else needs to change.

### Python

```python
from openai import OpenAI

# Point the openai client at LM Studio
client = OpenAI(
    base_url="http://localhost:1234/v1",
    api_key="lm-studio"   # ignored, can be anything
)

# Standard chat completion
response = client.chat.completions.create(
    model="qwen2.5-coder-7b-instruct",
    messages=[
        {"role": "system", "content": "You are a helpful coding assistant."},
        {"role": "user",   "content": "Write a Python function to reverse a linked list."}
    ],
    temperature=0.7,
    max_tokens=1024
)
print(response.choices[0].message.content)

# Streaming response
stream = client.chat.completions.create(
    model="qwen2.5-coder-7b-instruct",
    messages=[{"role": "user", "content": "Explain async/await in Python"}],
    stream=True
)
for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

---

### JavaScript / Node.js

```javascript
import OpenAI from 'openai';  // npm install openai

const client = new OpenAI({
  baseURL: 'http://localhost:1234/v1',
  apiKey:  'lm-studio',
});

// Chat completion
const completion = await client.chat.completions.create({
  model:    'qwen2.5-coder-7b-instruct',
  messages: [
    { role: 'system', content: 'You are a helpful coding assistant.' },
    { role: 'user',   content: 'Generate a REST API in Express.js' }
  ],
  temperature: 0.7,
});
console.log(completion.choices[0].message.content);

// Streaming
const stream = await client.chat.completions.create({
  model: 'qwen2.5-coder-7b-instruct',
  messages: [{ role: 'user', content: 'Write a quicksort in TypeScript' }],
  stream: true,
});
for await (const chunk of stream) {
  process.stdout.write(chunk.choices[0]?.delta?.content ?? '');
}
```

---

### C# (.NET)

```csharp
// dotnet add package OpenAI
using OpenAI;
using OpenAI.Chat;

var client = new OpenAIClient(
    new Uri("http://localhost:1234/v1"),
    new ApiKeyCredential("lm-studio")
);

var chatClient = client.GetChatClient("qwen2.5-coder-7b-instruct");

var response = await chatClient.CompleteChatAsync(
    new[] {
        new SystemChatMessage("You are a helpful coding assistant."),
        new UserChatMessage("How do I use LINQ in C#?"),
    }
);
Console.WriteLine(response.Value.Content[0].Text);
```

---

### Available API Endpoints

| Endpoint | Description |
|---|---|
| `GET /v1/models` | List loaded models |
| `POST /v1/chat/completions` | Chat (OpenAI-compatible) |
| `POST /v1/completions` | Raw text completion |
| `POST /v1/embeddings` | Generate embeddings (for RAG pipelines) |

---

## Resources

- **Official site:** https://lmstudio.ai
- **Docs:** https://docs.lmstudio.ai
- **GitHub:** https://github.com/lmstudio-ai
- **Discord:** https://discord.gg/aPQfnNkxGC
- **Model hub:** https://huggingface.co/models?library=gguf

---

*Guide covers LM Studio v0.3.x · Not affiliated with LM Studio AI*
