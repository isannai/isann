# Port policy

> How iSANN assigns ports across a node — engines, provider, broker.

## The rule

**Inside a container, a service always listens on `8080`.** Only the port it exposes to the host differs — which keeps the internal port uniform (no per-engine bookkeeping) while the host stays conflict-free.

## Host-facing ports

`isannd`, the provider, and the broker all run on the host and talk to each other over `127.0.0.1`. Two of them also serve a host port:

| Service | Host port | Used by |
|:--|:--|:--|
| **broker** | `8080` | browser (web UI), management |
| **provider** | `8090` | `isann infer` → your local `isannd` → provider |

They differ (`8080` vs `8090`) so both can run at once.

## Engines

Each engine container listens on `8080` internally and is published to the host on a **localhost-only** port:

| Engine | Host port (`127.0.0.1`) | → container |
|:--|:--|:--|
| sd | `7860` | `8080` |
| llama | `7862` | `8080` |
| vllm | `7864` | `8080` |

The **provider reaches each engine through its localhost port** — e.g. `127.0.0.1:7862` for llama. The provider runs on the host, so it dials these host-published ports (not the container's internal address).

> Engines also get static IPs on the shared `isann` docker network (sd `10.10.1.11`, vllm `10.10.1.12`, llama `10.10.1.13`) for container identity. The provider→engine path uses the localhost ports above, not these IPs — the provider is a host process, not on the docker network.

## Engines are never exposed to the network

**The provider is the only door to an engine** — and it's where every request is authenticated and metered/billed. Engine ports are bound to `127.0.0.1`, so they're **unreachable from the LAN or other nodes**: a remote caller can't hit an engine directly to **bypass payment**. It must go through the provider (`:8090`), which meters it.

So the localhost engine ports are not just for debugging — they're how the host-side provider reaches the engines, kept off the network on purpose.

To change an engine's host port, set `PORT` in that engine's `.env`. Leave the internal `8080` alone.

## Why same port inside, different outside?

- **Inside (`8080`)** — containers are isolated, so reusing one port everywhere removes per-engine config drift.
- **Outside (host)** — the host has a single network stack, so a host port can be bound by only one service at a time. Each engine gets a distinct host port (`7860`/`7862`/`7864`), and the two host-facing services use distinct ports (`8080`, `8090`).

---

← Back to **[README](../README.md)** · **[`isann` reference](../cli/cli-reference.md)**
