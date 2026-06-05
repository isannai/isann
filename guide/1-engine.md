# 1 · Install an engine

> Prerequisite: a working node — `ivm setup` done (WSL2 + Docker), `isannd` running (`ivm service start`), `isann docker status` shows **wsl + docker running**.

An **engine** is a containerized inference backend. `isannd` owns the Docker lifecycle; you drive it with `isann docker`. The shipped engines:

| Engine | Modality | Image |
|:--|:--|:--|
| `llama` | text / chat (llama.cpp, GGUF) | `ghcr.io/isannai/llama` |
| `sd` | image (Stable Diffusion) | `ghcr.io/isannai/sd` |
| `vllm` | text / chat (vLLM, high-throughput) | `vllm/vllm-openai` (upstream) |

## Create the engine

```cmd
isann docker create llama
```

This pulls the image (first time only), joins the shared `isann` Docker network, and **creates + starts** the container:

```console
$ isann docker create llama
 Image ghcr.io/isannai/llama:latest Pulling
 ...
[isann] llama: created and started (container: llama)
```

Verify:

```cmd
isann docker ps
```

## What's next

The container is **up but not yet serving** — `llama` needs a GGUF model, and the engine `.env` must point at it. If you start an engine with no model, the container exits (e.g. llama.cpp logs `No such file … .gguf`). That's expected at this stage.

Continue:

1. **[Install a model](2-model.md)** — download a model into the engine.
2. **[Configure the engine](3-config.md)** — point `.env` at the model + tune it.
3. **[Profiles](4-profile.md)** *(optional)* — save/switch `.env` presets.
4. **[Start the engine](5-start.md)** — apply config and verify it serves.
5. **[Run inference](6-inference.md)**.

## Lifecycle reference

| Command | Action |
|:--|:--|
| `isann docker create <engine>` | Pull image + create & start the container. |
| `isann docker ps` | List containers. |
| `isann docker start\|stop\|restart <name>` | Wake / graceful stop / stop+start. |
| `isann docker rm <name> [--force]` | Remove (running needs `--force`). |
| `isann docker pull <image>` | Pull/update an image without creating a container. |

Full flags: **[`isann` reference → docker](../cli/cli-reference.md)**.
