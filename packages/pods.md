# @mariozechner/pi-pods

**Version:** 0.65.2 · **License:** MIT · [npm](https://www.npmjs.com/package/@mariozechner/pi-pods)

CLI tool for deploying and managing vLLM inference servers on GPU cloud pods. Automatically sets up OpenAI-compatible API endpoints on DataCrunch, RunPod, Vast.ai, and similar providers.

---

## Installation

```bash
npm install -g @mariozechner/pi-pods
```

This installs the `pi` binary (separate from the coding-agent `pi` binary — use in different contexts).

---

## Quick Start

```bash
# 1. Set up a pod (first time)
pi pods setup my-pod "ssh root@1.2.3.4" --models-path /mnt/hf-models

# 2. Start a model on the pod
pi start qwen3-coder-30b

# 3. Chat with the model
pi agent qwen3-coder-30b "Explain how vLLM batches requests"

# 4. Check running models
pi list

# 5. Stop a model
pi stop qwen3-coder-30b
```

---

## Pod Setup

### DataCrunch (Recommended)

DataCrunch provides shared NFS storage, so models are downloaded once and shared across pods:

```bash
pi pods setup dc1 "ssh root@1.2.3.4" \
  --mount "sudo mount -t nfs nfs.fin-02.datacrunch.io:/models /mnt/hf-models"
```

The `--mount` command is run during setup to mount shared model storage. The models path is auto-extracted from the mount target.

### RunPod

RunPod provides persistent volumes:

```bash
pi pods setup runpod "ssh root@pod-id.runpod.io" \
  --models-path /runpod-volume
```

### Generic SSH Pod

Any Ubuntu machine accessible via SSH:

```bash
pi pods setup my-server "ssh -i ~/.ssh/id_ed25519 user@server.example.com" \
  --models-path /data/models
```

---

## Pod Management Commands

### `pi pods setup <name> <ssh-command>`

Set up vLLM on a fresh Ubuntu pod.

**Options:**
- `--mount <command>` — Run a mount command before setup (e.g., NFS mount)
- `--models-path <path>` — Where models are stored on the pod (default: `/mnt/hf-models`)
- `--vllm-version <version>` — `release` (default), `nightly`, or `gpt-oss`

**What setup does:**
1. Validates `HF_TOKEN` environment variable (required for model downloads)
2. Detects GPUs on the pod (name, memory)
3. Installs vLLM via pip
4. Configures the models path

### `pi pods active [name]`

List all pods or switch the active pod:

```bash
pi pods active          # List all pods
pi pods active my-pod   # Switch active pod
```

### `pi pods remove <name>`

Remove a pod from the configuration:

```bash
pi pods remove my-pod
```

### `pi shell`

Open an interactive SSH shell on the active pod:

```bash
pi shell
```

### `pi ssh <command>`

Run a command on the active pod:

```bash
pi ssh "nvidia-smi"
pi ssh "df -h"
```

---

## Model Commands

All model commands target the **active pod** unless `--pod <name>` is specified.

### `pi start <model-name>`

Start a vLLM inference server for a model.

```bash
pi start qwen3-coder-30b
pi start qwen3-coder-30b --gpus 2
pi start qwen3-coder-30b --memory 70% --context 32k
pi start kimi-k2 --vllm-version nightly
```

**Options:**
- `--gpus <n>` — Number of GPUs to use
- `--memory <percent>` — GPU memory pre-allocation (default: 50%)
- `--context <size>` — Max context: `4k`, `8k`, `16k`, `32k`, `64k`, `128k` (default: model max)
- `--vllm <extra-args>` — Additional vLLM arguments
- `--pod <name>` — Target a specific pod

**GPU allocation:** Uses round-robin across available GPUs. If a model needs 2 GPUs, pi assigns the next 2 unused ones.

**Port allocation:** Starts at 8001, increments for each model.

### `pi stop <model-name>`

Stop a running model:

```bash
pi stop qwen3-coder-30b
```

### `pi list`

Show all running models on the active pod:

```bash
pi list
# Output:
# MODEL               PORT  GPUS  STATUS
# qwen3-coder-30b     8001  0,1   running
# glm-4.5-air         8002  2     running
```

### `pi logs <model-name>`

Stream vLLM logs from the pod:

```bash
pi logs qwen3-coder-30b
```

---

## Interactive Agent

Use a running model for an interactive coding session:

```bash
# Start interactive session
pi agent qwen3-coder-30b

# Single query
pi agent qwen3-coder-30b "Review this Python function" --file src/main.py

# With session persistence
pi agent qwen3-coder-30b --session my-session "Continue our work"
```

### Agent Tools

The agent has access to:

| Tool | Description |
|------|-------------|
| `read` | Read file contents |
| `bash` | Execute shell commands locally |
| `list` | List directory contents |
| `glob` | File pattern matching |
| `rg` | Search file contents (ripgrep) |

Sessions are saved in `~/.pi/sessions/`.

---

## Predefined Model Configurations

Pi-pods ships optimized vLLM configurations for known agentic models:

### Qwen Family

| Model | GPUs | Notes |
|-------|------|-------|
| `qwen2.5-coder-32b` | 1–2 | hermes tool parser |
| `qwen3-coder-30b` | 1–2 | qwen3_coder tool parser |
| `qwen3-coder-30b-fp8` | 1 | FP8 quantized, ~30GB VRAM |
| `qwen3-coder-480b` | 8 | Tensor-parallel, H100/H200 |
| `qwen3-coder-480b-fp8` | 8 | Data-parallel mode |

### GPT-OSS Family

Requires `--vllm-version gpt-oss`:

| Model | GPUs | Notes |
|-------|------|-------|
| `gpt-oss-20b` | 1 | Async scheduling, `/v1/chat/completions` |
| `gpt-oss-120b` | 1–8 | Tool calling via `/v1/responses` only |

### GLM Family

| Model | GPUs | Notes |
|-------|------|-------|
| `glm-4.5` | 8–16 | glm45 parser + reasoning-parser, thinking mode |
| `glm-4.5-fp8` | 4–8 | FP8 quantized |
| `glm-4.5-air` | 1–2 | Smaller variant |
| `glm-4.5-air-fp8` | 1–2 | FP8 quantized |

### Kimi Family

| Model | GPUs | Notes |
|-------|------|-------|
| `kimi-k2` | 16 minimum | kimi_k2 tool parser, 128k context |

### Custom Model

Run any Hugging Face model not in the predefined list:

```bash
pi start my-model \
  --model "org/model-name-on-hf" \
  --gpus 4 \
  --vllm "--tensor-parallel-size 4 --tool-call-parser hermes"
```

---

## vLLM Versions

| Version | Use Case |
|---------|---------|
| `release` (default) | Stable release, recommended for most models |
| `nightly` | Latest features, required for some newer models (GLM-4.5) |
| `gpt-oss` | Special build for OpenAI GPT-OSS models only |

Specify during pod setup:
```bash
pi pods setup my-pod "ssh root@1.2.3.4" --vllm-version nightly
```

Or per-model start:
```bash
pi start glm-4.5 --vllm-version nightly
```

---

## Configuration

Pi-pods stores configuration in `~/.pi/pods.json` (override with `PI_CONFIG_DIR`):

```json
{
  "pods": {
    "my-pod": {
      "ssh": "ssh root@1.2.3.4",
      "gpus": [
        { "id": 0, "name": "NVIDIA H200 SXM", "memory": "141GB" },
        { "id": 1, "name": "NVIDIA H200 SXM", "memory": "141GB" }
      ],
      "models": {
        "qwen3-coder-30b": {
          "model": "Qwen/Qwen3-Coder-30B-Instruct",
          "port": 8001,
          "gpu": [0, 1],
          "pid": 12345
        }
      },
      "modelsPath": "/mnt/hf-models",
      "vllmVersion": "release"
    }
  },
  "active": "my-pod"
}
```

---

## Environment Variables

| Variable | Description |
|----------|-------------|
| `HF_TOKEN` | Hugging Face token (required for gated models) |
| `PI_CONFIG_DIR` | Override config directory (default: `~/.pi`) |

---

## API Endpoints

Each started model exposes an **OpenAI-compatible API**:

```
http://<pod-ip>:<port>/v1/chat/completions
http://<pod-ip>:<port>/v1/models
```

Use with any OpenAI-compatible client:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://pod-ip:8001/v1",
    api_key="not-needed"
)

response = client.chat.completions.create(
    model="Qwen/Qwen3-Coder-30B-Instruct",
    messages=[{"role": "user", "content": "Hello"}]
)
```

**Note:** GPT-OSS-120B uses `/v1/responses` for tool calling (not `/v1/chat/completions`).

---

## Integration with Pi

Pi-pods models can be used directly in the pi coding agent via custom provider registration:

```typescript
import { registerApiProvider } from "@mariozechner/pi-ai";

registerApiProvider({
  api: "openai-completions",
  streamSimple: myCustomStreamSimple
}, "my-vllm-pod");
```

Or configure in `~/.pi/agent/models.json`:
```json
{
  "models": [
    {
      "id": "qwen3-coder-30b",
      "name": "Qwen3 Coder 30B",
      "baseUrl": "http://pod-ip:8001/v1",
      "api": "openai-completions"
    }
  ]
}
```
