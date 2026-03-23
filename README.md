# Local AI stack

A super lightweight and fast environment for running programming LLMs (such as `Qwen2.5-Coder`) locally on your Mac, integrated directly into your terminal via OpenCode.

## 🪄 Discovering the Best Model for Your Machine

The choice of which model version to use (7B, 14B, 30B...) depends exclusively on the amount of Unified Memory (RAM/VRAM) your computer has.

To find out exactly what the "sweet spot" is for your hardware, **we highly recommend using [llmfit](https://github.com/AlexsJones/llmfit)**.

Run in your terminal:
```bash
npx llmfit
```

`llmfit` will perform a quick analysis of your computer's hardware and recommend the exact `.gguf` file quantization that will run smoothly without choking your development environment (Example common suggestions for the 16GB M2: `q4_k_m` or `q5_k_m` in the 7B and 14B variants).

## ⚙️ Centralized Configuration

1. At the root of the project, you will see a file called `.env.example`. Copy it or simply rename it to `.env`:
```bash
cp .env.example .env
```
2. Open your newly created `.env` file.
3. Replace the `MODEL_FILE` variable value exactly with the name that **`llmfit`** recommended during its inspection:
```env
MODEL_FILE="qwen2.5-coder-14b-instruct-q4_k_m.gguf"
```
*(From this point forward, all project scripts will read from this single "source of truth").*

4. Download the official repository file to your machine using our robust bash script:
```bash
./scripts/download-model.sh
```

## 🚀 Daily Usage

### Start Everything (Local Server + Terminal Chat)
Starts the `llama.cpp` engine server invisibly in the background and then immediately opens the OpenCode CLI.
```bash
./scripts/start-all.sh
```

### Stop Everything
Safely shuts down the AI server and completely frees up your RAM.
```bash
./scripts/stop-all.sh
```

### Sync Models with OpenCode
Updates `~/.config/opencode/opencode.json` based on running servers and available models:
```bash
./scripts/sync-opencode-models.sh
```

### Start Only the Local AI Server
Useful if you want to connect the AI to your native VS Code, Cursor, or Cline extensions via port `8080`.
```bash
./scripts/start-llama.sh
```

## 📋 Diagnostics and Performance Logs

To track in real-time how hard the machine is working (tokens/s, processing, and temps):
```bash
tail -f logs/llama.log
```

To verify in the raw interface if the local API is online and compatible with the OpenAI standard:
```bash
curl http://localhost:8080/v1/models
```

## 📐 Choosing the Right Context Size (LLAMA_CTX)

The context window (`LLAMA_CTX`) is the total number of tokens the model can hold in memory at once — including the system prompt, tool definitions, conversation history, and your message. Getting this wrong causes two symptoms:

- **Too small** → `request (N tokens) exceeds the available context size` errors, or the client constantly compacting/summarizing the conversation
- **Too large** → Metal OOM crashes (`kIOGPUCommandBufferCallbackErrorOutOfMemory`)

### How to estimate the minimum context you need

Coding assistants like OpenCode send far more than your typed message. A single request includes:

| Component | Approx. tokens |
|---|---|
| System prompt + tool definitions | 4 000 – 10 000 |
| Conversation history | grows over time |
| Your message | 10 – 500 |
| **Minimum safe floor** | **~16 000** |

**Rule of thumb: start at 16 384 for coding assistants.** If you still see the "exceeds context" error, check the exact token count in the error message and set `LLAMA_CTX` to the next power of 2 above it (e.g. 24 576, 32 768).

### How to check if your RAM can handle it

On Apple Silicon, GPU and CPU share unified memory, so the formula is:

```
model_weights_GB + kv_cache_GB + ~2 GB OS headroom ≤ total RAM
```

Rough KV cache estimate: `LLAMA_CTX × 0.0002 GB per billion parameters`

Examples for a 24 GB M4:

| Model | Weights | CTX 16k KV cache | Total | Fits? |
|---|---|---|---|---|
| Qwen 9B Q8_0 | ~9 GB | ~1.5 GB | ~12.5 GB | ✅ |
| Qwen 14B Q4_K_M | ~8 GB | ~2.3 GB | ~12.3 GB | ✅ |
| LFM2 24B Q8_0 | ~24 GB | ~4 GB | ~30 GB | ❌ OOM |
| LFM2 24B Q4_K_M | ~13 GB | ~4 GB | ~19 GB | ✅ |

### Adjusting the context

All tuning is done in `.env` — never edit `start-llama.sh` directly:

```env
LLAMA_CTX=16384   # safe default for coding assistants
```

Then restart: `./scripts/start-llama.sh restart`

## 💡 Notes (Troubleshooting)

**If memory gets too full (OOM crashes):**
Reduce `LLAMA_CTX` first, then `LLAMA_NGL`. Edit `.env`:
```env
LLAMA_CTX=8192
LLAMA_NGL=40
```

**If word generation is too slow:**
On Apple Silicon, make sure all layers are on the GPU:
```env
LLAMA_NGL=99
```
You can also try enabling flash attention (`LLAMA_FA=on`) for a speed boost once the model is stable.

**If OpenCode text is black / hard to read on a dark terminal:**
While running OpenCode, simply type `/theme` and press **Enter** to select a dark-mode friendly theme (like `system`, `tokyonight`, or `one-dark`).
