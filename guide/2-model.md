# 2 · Install a model

> Prerequisite: an engine created — see **[1 · Install an engine](1-engine.md)**.

Models are downloaded **per engine** into that engine's `models/` directory. `isann model pull` streams the download (resumable, hash-verified) and works for HuggingFace and Civitai URLs, or a local `file://` path.

## Pull a model

```
isann model pull <URL> --engine <e> --kind <model|lora|vae> --name <name> [--arch <a>] [--force]
```

| Flag | Meaning |
|:--|:--|
| `--engine` | Target engine (`llama`, `sd`, …). |
| `--kind` | `model` (checkpoint), `lora`, or `vae`. |
| `--name` | Directory name to store it under (you reference this in the engine `.env`). |
| `--arch` | SD only — `sd15` / `sdxl` / … (selects the sub-folder). |
| `--force` | Re-download / overwrite. |

### Example — a GGUF model for `llama`

```console
$ isann model pull "https://huggingface.co/bartowski/Qwen2.5-14B-Instruct-GGUF/resolve/main/Qwen2.5-14B-Instruct-Q4_K_M.gguf" \
    --engine llama --kind model --name Qwen2.5-14B-Instruct-Q4_K_M
  Qwen2.5-14B-Instruct-Q4_K_M.gguf  42.1%  (3.8 / 9.0 GB)
[isann] done -- engines/llama/models/defaults/Qwen2.5-14B-Instruct-Q4_K_M
  hash: sha256:abc123...
```

`llama` loads a **single GGUF file**. Pick a quant that fits your VRAM (e.g. a 14B `Q4_K_M` ≈ 9 GB needs ~10 GB+ to fully offload; smaller quants / 7B models for smaller GPUs).

> **Tip:** `isann model search "<query>" --source hf --kind model` finds models and their URLs; `isann model info <URL>` shows file sizes + hashes before you download.

## Manage models

| Command | Description |
|:--|:--|
| `isann model list [--engine e] [--kind k]` | Installed models / loras / vaes. |
| `isann model search <q>` · `--hash <sha>` | Search Civitai / HF / local. |
| `isann model info <URL>` | Files (size + sha256), total size, kind, arch. |
| `isann model rm --engine e --kind k --name n [--force]` | Delete an installed model. |
| `isann model register --engine e --kind k --name n` | Hash an already-on-disk model + write `package.json` (no download). |

`pull` / `rm` / `list` accept `--nodes` for cross-node operation (multi-node parallel download dashboard).

## Next

The file is on disk, but the engine doesn't know to load it yet → **[3 · Configure the engine](3-config.md)** (set `MODEL=` in the `.env`).
