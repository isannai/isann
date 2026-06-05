# 3 · Configure the engine

> Prerequisite: an engine created + a model downloaded — see **[1](1-engine.md)**, **[2](2-model.md)**.

Each engine is **self-contained** under `<install_root>/engines/<engine>/`:

```
engines/llama/
├── docker-compose.yml
├── .env                  # ← you edit this
├── models/               # models you pulled (step 2)
└── scripts/run.sh        # container entrypoint
```

The `.env` is read by `docker compose` when `isannd` (re)starts the container. **All paths are `./` relative** to the engine directory, so the engine folder is portable across machines and install roots — never put absolute paths here.

## Point the engine at your model

For `llama`, the one required field is **`MODEL`** — the GGUF path *relative to* `MODELS_DIR`:

```ini
# engines/llama/.env
MODELS_DIR=./models/defaults
MODEL=Qwen2.5-14B-Instruct-Q4_K_M/Qwen2.5-14B-Instruct-Q4_K_M.gguf
```

`<name>/<file>.gguf` = the `--name` you pulled with, then the GGUF filename inside it. Inside the container the engine loads `/models/$MODEL`.

## Common fields (llama)

| Field | Meaning |
|:--|:--|
| `MODEL` | GGUF path under `MODELS_DIR` (required). |
| `MODELS_DIR` | Models directory (relative, e.g. `./models/defaults`). |
| `CTX_SIZE` | Context window (tokens). |
| `GPU_LAYERS` | Layers offloaded to GPU (lower it if VRAM is tight). |
| `SERVED_MODEL_NAME` | Optional alias the API reports. |
| `IMAGE_NAME` / `IMAGE_TAG` | Engine image — defaults `ghcr.io/isannai/llama` / `latest`. |
| `IP` | Static IP on the `isann` network (e.g. `10.10.1.13`) — don't change unless you know the layout. |

> **Image registry:** engine images live on **GHCR** (`ghcr.io/isannai/llama`, `ghcr.io/isannai/sd`). The registry prefix is required — a bare `isannai/llama` resolves to Docker Hub and fails to pull. `vllm` uses the upstream `vllm/vllm-openai`.

Other engines (`sd`) use the same pattern with their own fields (`MODELS_DIR=./models/sd15/defaults`, VAE/LoRA dirs, etc.).

## Apply changes

`.env` changes only take effect on a container (re)start:

```cmd
isann docker restart llama
```

> Editing `.env` by hand is fine for a single config. To keep **named, switchable** configs (e.g. `lowmem` vs `highctx`), use **[4 · Profiles](4-profile.md)** instead.

## Next

→ **[4 · Profiles](4-profile.md)** (optional) or jump to **[5 · Start the engine](5-start.md)**.
