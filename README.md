# iSANN

### interStellar Artificial Neural Network

> A peer-to-peer platform for building and sharing your **own** personal AI — decentralized AI (**deAI**).

Bundle your own machines into a personal AI node — run it yourself today.

**Components**

| Binary | Role |
|:--|:--|
| **`ivm`** | Installer / version manager — the tool you start with |
| **`isannd`** | The node daemon — control plane, Docker/engine, P2P transport |
| **`isann`** | CLI client that talks to your local `isannd` |

## Versioning

Releases are numbered `x.y.z`:

| Part | Meaning |
|:--|:--|
| **`x`** — major | May change configuration or break compatibility between versions. |
| **`y`** — minor | **Odd = development**, **even = stable**. An even `y` is the stabilized release of the odd `y` directly below it (e.g. `0.2.x` is the stable line for the `0.1.x` dev series). |
| **`z`** — patch | Bug fixes. |

For production, run a **stable** (even-`y`) release. The current `0.1.x` line is an early **development** series.

---

## Getting Started

**Requirements:** an NVIDIA GPU (driver ≥ 525 / CUDA 12.0). `ivm setup` installs the rest — WSL2 + Docker on Windows, Docker + NVIDIA Container Toolkit on Linux.

> Download `ivm-<os>-amd64.zip` from **Releases**, then follow your platform below.
> `--version` matches the release tag **exactly** — copy it from the **Releases** page (e.g. `0.1.2`).

### Windows

Unzip `ivm-windows-amd64.zip` to your install root (e.g. `D:\iann`), then:

```cmd
.\ivm init          :: TLS cert + layout + PATH; writes activate.bat
call activate       :: load PATH into THIS shell so ivm/isann/isannd resolve
                    ::   (or just open a new terminal instead)
ivm check           :: detect WSL2 / Docker / toolkit / driver (read-only)
ivm setup           :: UAC -> install WSL2 + Ubuntu + Docker  (reboot if prompted)
ivm install         :: fetch the latest node suite (isann/isannd/proxy/fetcher + conf/engines)

ivm service install :: register isannd (UAC -> on-demand Scheduled Task, no password)
ivm service start
ivm service status  :: Running

isann account create --alias me
isann auth transfer --owner me     :: register the first owner
isann info
isann docker status                :: wsl + docker running
```

The node runs as a Scheduled Task under your account: it **survives logoff** (Session 0), stores **no password** (S4U), and does **not** auto-start at boot. Prefer a console? Just run `bin\isannd.bat`.

### Linux

Unzip `ivm-linux-amd64.zip` to your install root (e.g. `/opt/iann`), then:

```bash
./ivm init          # TLS cert + layout + PATH (~/.bashrc); writes ./activate
source ./activate   # load PATH into this shell  (or open a new shell instead)
ivm check           # detect docker / nvidia-container-toolkit / driver
ivm setup           # sudo -> install Docker + NVIDIA Container Toolkit
ivm install         # fetch the latest node suite

ivm service install # systemd unit (sudo)
ivm service start

isann account create --alias me
isann auth transfer --owner me
isann info
```

> The packaged `ivm-linux` zip is still being finalized — check the Releases page for availability.

### Updating

```console
ivm list                     # local cache + active version
ivm list -remote             # available releases on GitHub
ivm install --version 0.1.2
ivm use --version 0.1.2       # stop -> swap binaries -> start (one shot)
```

---

## Operating your node

With the node up (`isann docker status` → **docker running**), set up an engine and serve a model. Follow these in order:

1. **[Install an engine](guide/1-engine.md)** — create a containerized inference backend (`llama` / `sd` / `vllm`).
2. **[Install a model](guide/2-model.md)** — download a model into the engine.
3. **[Configure the engine](guide/3-config.md)** — point the engine `.env` at your model + tune it.
4. **[Profiles](guide/4-profile.md)** *(optional)* — save and switch named `.env` configs.
5. **[Start the engine](guide/5-start.md)** — apply the config and verify it's serving.
6. **[Run inference](guide/6-inference.md)** — locally, or on another node with `--nodes`.

---

## CLI reference

- **[`ivm` commands](cli/ivm-reference.md)** — node lifecycle: `install` · `switch` · `service` · `use` · `list` · `rm`
- **[`isann` commands](cli/cli-reference.md)** — node client: `account` · `auth` · `docker` · `infer` · `model` · `profile` · discovery

## Roadmap

- **[Full roadmap](roadmap/roadmap.md)** — quarterly milestones through 2027: launch → Web3 payment rails → marketplace → platform reach.

---

*Source repository is private; this release repo hosts the binaries + this guide. Full documentation (inference, node operation) is in progress.*

> **Roadmap** — node sharing and metered payment between nodes (decentralized AI on `ERC-8004` identity / `ERC-8183` escrow / `x402` payment, no proprietary token) land in **Phase 2 (Dec 2026 – Feb 2027)**. See the **[full roadmap](roadmap/roadmap.md)**.
