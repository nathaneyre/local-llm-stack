# Local LLM Stack

Self-hosted Ollama + Open WebUI, providing:

- **Quick Q&A** via Open WebUI (browser chat interface)
- **Coding assistance** in VS Code (via Continue.dev, Cline, or similar extensions)

This project is infrastructure only — it runs standalone and persistently in the background. It has no knowledge of, or dependency on, any individual app project. Application projects connect *to* this stack over the network; they don't live inside it.

---

## Prerequisites

- GPU Driver is up to date
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) is installed and:
  - Uses the settings:
    - General
      - Enable "Start Docker Desktop when you sign in to your computer"
      - Disable "Open Docker Dashboard when Docker Desktop starts"
    - Settings | WSL Integration
      - Enable "Enable integration with my default WSL distro"
  - Verify GPU is visible to Docker: `docker run --rm --gpus=all nvidia/cuda:12.4.1-base-ubuntu22.04 nvidia-smi`
- ~20GB+ free disk space per model pulled
- This folder is cloned locally: `git clone git@github.com-ne:nathaneyre/local-llm-stack.git`

---

## Directory structure

```
local-llm-stack/
├── docker-compose.yml
├── modelfiles/
│   └── model-name-custom.Modelfile
└── .env                              # secrets (gitignored)
```

---

## Quick start

```bash

# 1. Bring up the stack
docker compose up -d

# 2. Pull the base model
docker exec -it ollama ollama pull model_name:tag

# 3. Build the custom model (with your system prompt baked in)
docker exec -it ollama ollama create model-name-custom -f /modelfiles/model-name-custom.Modelfile

# 4. Verify
curl http://localhost:11434/api/tags
```

Open WebUI: `http://localhost:3080`
Ollama API: `http://localhost:11434`

---

## Custom system prompt

Every model that should honor your standing system prompt is built from a **Modelfile** in `modelfiles/`, rather than set ad hoc per-client. This keeps the prompt versioned, reproducible, and consistent across Open WebUI, VS Code, and any raw API calls.

`modelfiles/model-name-custom.Modelfile`:
```dockerfile
FROM model_name:tag
SYSTEM """
<your system prompt text here>
"""
PARAMETER num_ctx 65536
```

To update the prompt: edit the Modelfile, then rebuild:
```bash
docker exec ollama ollama create model-name-custom -f /modelfiles/model-name-custom.Modelfile
```

Point all clients (Open WebUI model selector, VS Code extension config, API calls) at `model-name-custom` — never the bare base tag — so the system prompt is guaranteed to apply everywhere.

---

## Connecting from app projects / devcontainers

App projects (Laravel, React, etc.) run in their own devcontainers, entirely separate from this stack. They reach Ollama over the network rather than sharing a filesystem or lifecycle with it.

Any container on Docker Desktop/WSL2 can reach this stack via:
```
http://host.docker.internal:11434
```

Any host tooling can reach this stack via:
```
http://localhost:11434
```

No shared Docker network required — works as long as this stack is running.

Use `model-name-custom` (not the base tag) as the model name in any client config, so the system prompt applies.

---

## Maintenance

- **List models:** `docker exec -it ollama ollama list`
- **Remove models:** `docker exec -it ollama ollama rm model_name:tag`
- **Update models:** `docker exec ollama ollama pull model_name:tag` periodically; rebuild custom variants afterward.
- **Pin image tags** in `docker-compose.yml` (avoid `:latest`) and bump deliberately, checking release notes first.
- **Check VRAM headroom** if you raise `OLLAMA_CONTEXT_LENGTH` — KV cache scales with context length and can push a model past 24GB even if the base weights fit comfortably.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Model load OOMs or spills to system RAM | Context length + KV cache too large for remaining VRAM after model weights. Lower `OLLAMA_CONTEXT_LENGTH` or use a quantized KV cache type. |
| Devcontainer can't reach Ollama | Confirm this stack is running (`docker ps`), confirm `host.docker.internal` resolves inside the app container (Linux hosts may need an explicit `extra_hosts` entry). |
| Slow response switching between models | Expected — Ollama unloads/reloads models on demand. Keep `OLLAMA_MAX_LOADED_MODELS` and usage patterns in mind if this matters to you. |