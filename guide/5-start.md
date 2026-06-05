# 5 · Start the engine

> Prerequisite: engine created, model installed, `.env` pointing at the model — **[1](1-engine.md)**–**[3](3-config.md)** (and **[4](4-profile.md)** if you use profiles).

`isann docker create` already started the container, but it had no model/config yet. Now that the model and `.env` are in place, **restart** so the engine loads them:

```cmd
isann docker restart llama
```

## Verify it's serving

```cmd
isann docker ps                 :: container is "running"
isann docker probe llama        :: engine HTTP readiness (does not wake WSL)
```

`probe` returning ready means the engine loaded the model and is accepting requests. You're ready to **[run inference](6-inference.md)**.

## Troubleshooting

| Symptom | Likely cause / fix |
|:--|:--|
| Container exits immediately; logs show `No such file … .gguf` | `MODEL` in `.env` doesn't match an installed file. Re-check **[3 · Configure](3-config.md)** and that the model from **[2](2-model.md)** is present (`isann model list --engine llama`). |
| `pull access denied … repository does not exist` on create | Image registry prefix missing — `IMAGE_NAME` must be `ghcr.io/isannai/<engine>`, not a bare `isannai/<engine>`. |
| `no configured subnet contains IP address 10.10.1.x` | The shared `isann` network has the wrong subnet. Recreate it: `docker network rm isann` then `docker network create --subnet 10.10.1.0/24 --gateway 10.10.1.1 isann`. (New `isannd` builds create it correctly.) |
| Slow / runs on CPU | Model too large for VRAM — lower `GPU_LAYERS` or use a smaller quant / model (**[2](2-model.md)**). |

Inspect logs / state with `isann docker inspect <name>`, and engine logs via Docker on the WSL/Linux side.

## Next

→ **[6 · Run inference](6-inference.md)**.
