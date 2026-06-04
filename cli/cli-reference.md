# isann CLI Reference

[← Back to README](../README.md) · [ivm reference](ivm-reference.md)

> **isann** is a thin client to your local `isannd` over loopback (`https://127.0.0.1:8443`). Each command lists the isannd **API** it maps to.

## Legend

- **API**: the mapped isannd endpoint `<METHOD> /internal/api/<path>`.
  - `local` = no isannd call (local files / wallet) · `GitHub API` · `gate` (reverse-proxied to a gate)
- **Nodes**: `--nodes` cross-node supported (✅) / not (—).
- **Status**: ✅ implemented · ⚠️ implemented but differs from design / not wired.

## Common mechanics

1. **Output flags** (all read commands): `-json` (raw), `-pretty` (indent), `--proj <dot.paths>` (field selection, MongoDB-projection style). isannd shapes the response server-side, so the HTTP API behaves the same.
2. **Session**: most commands require a prior `isann auth unlock` (the `.isann/session` token is auto-attached as `X-ISANN-Session`).
3. **Cross-node forward**: a `--nodes` command always wires through **`POST /internal/api/nodes/forward`** — the `adminPath` (e.g. `/rv/add`, `/docker/ps`) is carried in the body: local isannd → peer `/admin/<path>` → peer's own `/internal/api/<path>` loopback. The "API" below is the **local (single-node)** path; cross-node carries the same sub-path in the forward.
   - ⚠️ Query-string options are **not** forwarded: `auth list --roles`, `docker shutdown --force`, `profile rm --force`. (Body options like `--timeout` are fine.)

> Node bootstrap is split out — layout/cert = **`ivm init`**, first owner registration = **`isann auth transfer --owner`**. `isann init` has been **removed**.

---

## info / version

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann info` | Node version / OS / GPU / node_id (2-column). | `GET /info` | ✅ |
| `isann version` | CLI build version + local isannd version (2 lines). | `GET /version` (optional) | — |

```console
$ isann info
version   0.1.2
os        windows
node_id   0xabc...def
gpu       NVIDIA GeForce GTX 1650 (driver 555, CUDA 12.5)

$ isann version
isann CLI  0.1.2
isannd     0.1.2 (running at https://127.0.0.1:8443)
```

## account

Pure local — calls `pkg/wallet` directly, never touches isannd. `--nodes` not supported.

| Command | Description | API |
|---|---|---|
| `isann account create --alias n [--force]` | New keystore + register alias → address. Passphrase ≥ 8 chars. | `local` |
| `isann account import <path> --alias n` · `--pk <hex> --alias n` | Hardlink an existing keystore, or encrypt a raw 0x-hex private key. | `local` |
| `isann account list [-json]` | Alias / address / file / owner-role table. | `local` |
| `isann account pk --alias a` | Decrypt + print the raw private key (passphrase prompt + warning). | `local` |
| `isann account rm --alias a [-y]` | Remove a keystore (owner key protected — transfer first). | `local` |

```console
$ isann account create --alias office
[isann] generated 0xab12...cd34
[isann] alias "office" -> 0xab12...cd34
```

## auth

Session (unlock/lock/status) + roles (transfer/add/rm/list). transfer/add/rm/list support `--nodes`.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann auth unlock --account a [--passphrase p\|--passphrase-stdin] [--timeout s]` | Decrypt the key in isannd → session token + `.isann/session`. | `POST /auth/unlock` | — |
| `isann auth lock` | Revoke session + clear local token (idempotent). | `DELETE /auth/session` | — |
| `isann auth status [-json]` | Current session state (wallet / expiry / TTL). | `GET /auth/status` | — |
| `isann auth transfer --owner a [-y]` | Owner unset → first registration (local write); set → transfer (irreversible). | `PUT /auth/owner` (transfer only) | ✅ |
| `isann auth add --admin\|--user\|--issuer a` | Grant a role. | `POST /auth/{admins\|users\|issuers}` | ✅ |
| `isann auth rm --admin\|--user\|--issuer a` | Revoke a role. | `DELETE /auth/{plural}/<addr>` | ✅ |
| `isann auth list [--roles a,b]` | Role-holder list. | `GET /auth/config` | ✅ |

```console
$ isann auth unlock --account office
[isann] unlocked 0xab12...cd34
[isann] session expires in 1h0m0s

$ isann auth transfer --owner me      # owner unset -> first registration
[isann] owner registered -> 0x8fc2...9091
```

Passphrase priority for `unlock`: `--passphrase-stdin` > `ISANN_PASSPHRASE` > `--passphrase` > prompt.

## docker

Container preflight + lifecycle. **isannd owns the Docker lifecycle.** Cross-node: status/warmup/shutdown/ps/start/stop/restart. Not: create/rm/inspect/probe/pull.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann docker status` | WSL/docker readiness (side-effect free). | `GET /docker/status` | ✅ |
| `isann docker warmup` | Boot WSL+docker in the background (fire-and-forget). | `POST /docker/warmup` | ✅ |
| `isann docker shutdown [--force]` | `wsl --shutdown` (Windows only). | `POST /docker/shutdown [?force=1]` | ✅ |
| `isann docker ps` | Container list. | `GET /docker/ps` | ✅ (`<id>@<ip:port>`) |
| `isann docker create <engine>` | Compose-based spawn (streaming). | `POST /docker/create` | — |
| `isann docker start\|stop\|restart <name> [--timeout s]` | wake / graceful stop (+SIGKILL) / stop+start. | `POST /docker/{start\|stop\|restart}/<name>` | ✅ |
| `isann docker rm <name> [--force]` | Remove a container (running needs `--force`). | `DELETE /docker/rm/<name> [?force=1]` | — |
| `isann docker inspect <name>` | Raw `docker inspect` JSON. | `GET /docker/inspect/<name>` | — |
| `isann docker probe <name>` | Engine HTTP readiness (does not wake WSL). | `GET /docker/status` + `…/probe/<name>` | — |
| `isann docker pull <image>` | Image pull (streaming). | `POST /docker/pull` | — |

```console
$ isann docker status
COMPONENT  STATE    DETAIL
wsl        running  Ubuntu-22.04
docker     running  wsl, engine 27.1.1

$ isann docker create sd
Pulling sd ... done
[isann] sd: created and started (container: isann-sd)
```

## favorite

Bookmark node IDs (`alias → role:0xaddr`). `favorites.json` is owned by isannd; session required. `--nodes` not supported.

| Command | Description | API |
|---|---|---|
| `isann favorite add --alias n --nodeid <role:0xaddr> [--force]` | Register a bookmark (e.g. `P:0xab...`). | `POST /favorite/add` |
| `isann favorite list [-json]` | alias → node id table. | `GET /favorite/list` |
| `isann favorite rm --alias n` | Remove by alias. | `DELETE /favorite/<alias>` |

## ghcr

GHCR lookups. `list` = gate DB (reverse-proxy), `tags` = ghcr.io OCI live (anonymous). `--nodes` not supported.

| Command | Description | API |
|---|---|---|
| `isann ghcr list [--namespace ns]` | Images registered in the gate DB. | `GET /internal/gate/v1/ghcr/list` |
| `isann ghcr tags <image> [--limit N]` | Live tag list for a public image (`<owner>/<repo>`). | `GET /ghcr/tags?image=<image>` |

## infer

Inference jobs. **Different namespace** — not an `/internal/api/...` admin path but isannd's `/internal/api/infer` reverse proxy (or `/node/p:<id>/svc/*` with `--nodes`, or the provider directly with `--provider`).

`{base}` = `providerBaseURL()`: local `<isanndURL>/internal/api/infer` · `--nodes <id>` → `<isanndURL>/node/p:<id>` · `--provider <h:p>` → provider direct.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann infer run --engine e [--<param>..] [-wait] [-stdin] [--out f] [--preset n] [--nodes id] [--provider h:p]` | Submit a job (job_id; `-wait` polls + renders). | `POST {base}/svc/<svc>/v1/jobs` | ✅ |
| `isann infer chat --engine e [--system s] [--temperature f] [--max-tokens n]` | Multi-turn REPL (`/reset`, `/exit`). | `POST {base}/svc/<svc>/v1/jobs` per turn | ✅ |
| `isann infer status <id>` | Job status. | `GET {base}/v1/jobs/<id>` | ✅ |
| `isann infer result <id> [--out f] [-consume]` | Finished result (text→stdout, image/audio→file). | `GET {base}/v1/jobs/<id>/result` | ✅ |
| `isann infer rm <id>` | Delete a finished job. | `DELETE {base}/v1/jobs/<id>` | ✅ |
| `isann infer queue --engine e` | Service queue stats. | `GET {base}/v1/queue/stats?service=<svc>` | ✅ |

```console
$ isann infer run --engine text --prompt "Summarize in one line: ..." -wait
job_id: j-7f3a    service: llm-api    queue: 0/8
The summarized text appears here.

$ isann infer result j-9b1 --out out.png
saved out.png (2451233 bytes)
```

For `result`, `--proj` extracts a field — e.g. `--proj choices[0].message.content`. `infer` shapes output client-side.

## list

Read-only views. models/loras/vaes/profiles support `--nodes`. nodes/metrics query the RV (node-agnostic). favorites/rvs/accounts/presets delegate to other namespaces.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann list nodes [--rv r]` | Nodes registered on an RV. | `GET /list/nodes?rv=<url>` | — |
| `isann list metrics [--rv r]` | Per-(node,service) metrics from the RV. | `GET /list/metrics?rv=<url>` | — |
| `isann list models\|loras\|vaes [--engine e]` | Scan `engines/<e>/models/` (loras/vaes are KIND-filtered views). | `GET /list/models [?engine=<e>]` | ✅ |
| `isann list profiles [--engine e]` | Scan `engines/<e>/profiles/*.env`. | `GET /list/profiles [?engine=<e>]` | ✅ |
| `isann list favorites \| rvs \| accounts \| presets` | Delegate to favorite / rv / account / preset. | (delegated) | partial |

```console
$ isann list nodes --rv office
ID          ROLE      ADDR          OWNER     TPM
p:0xabc...  provider  1.2.3.4:7443  0xabc...  verified

$ isann list models --engine llama
ENGINE  KIND   TYPE    REG  NAME                 SIZE
llama   model  single   v   Qwen2.5-1.5B-Q4_K_M  1.0 GiB
```

## model

Model management. pull/rm/list support `--nodes`; info/search/register do not.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann model list [--engine e] [--kind k]` | Local cached models/loras/vaes. | `GET /list/models [?engine=<e>]` | ✅ |
| `isann model info <URL>` | Deep metadata: files (size+sha256), total size, kind, arch (HF + Civitai). | `GET /model/info?url=<URL>` | — |
| `isann model search <q> \| --hash <sha> [--source s] [--kind k] [--limit N]` | Search Civitai / HF / local. | `GET /model/search?...` | — |
| `isann model pull <URL> --engine e --kind k --name n [--arch a] [--force] [--nodes a,b]` | Download a model (NDJSON stream; multi-node parallel dashboard). | `POST /model/pull` | ✅ |
| `isann model register --engine e --kind k --name n [--force]` | Hash an on-disk model + write `package.json` (no download). | `POST /model/register` | — |
| `isann model rm --engine e --kind k --name n [--force]` | Delete an installed model directory. | `POST /model/rm` | ✅ |

```console
$ isann model pull https://huggingface.co/Qwen/Qwen2.5-1.5B --engine llama --kind model --name Qwen2.5-1.5B
  model.safetensors  42.18%  (442368000 / 1048576000 bytes)
[isann] done -- engines/llama/models/Qwen2.5-1.5B
  hash: sha256:abc123...   model_kind: single
```

## preset

Pure local — reads/writes `.isann/presets/<type>.json`, no isannd. `--nodes` not supported.

| Command | Description |
|---|---|
| `isann preset set --type t --name n key=value ...` | Create / merge a preset (value types auto-inferred). | 
| `isann preset list [--type t]` · `show --type t --name n` · `rm --type t --name n [-y]` | List / print / remove. |

```console
$ isann preset set --type text --name creative temperature=1.1 top_p=0.95
[isann] preset text/creative created (temperature=1.1, top_p=0.95)
```

## profile

Engine `.env` profiles. All leaves support `--nodes`. `profile rm --force` cannot combine with `--nodes` (query-string not forwarded).

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann profile list [--engine e]` | Profile list. | `GET /list/profiles [?engine=<e>]` | ✅ |
| `isann profile show --engine e --name n` | Dump a profile's `.env`. | `GET /profile/<e>/<n>` | ✅ |
| `isann profile use --engine e --name n` | Set the active profile (copies to `.env`). | `POST /profile/use` | ✅ |
| `isann profile create --engine e --name n [-y] [--force]` | New profile. | `POST /profile/create` | ✅ |
| `isann profile update --engine e --name n` | Edit a profile (current values as defaults). | `PUT /profile/update` | ✅ |
| `isann profile copy --engine e --from a --to b` | Clone a profile. | `POST /profile/copy` | ✅ |
| `isann profile rm --engine e --name n [-y] [--force]` | Delete a profile (active needs `--force`). | `DELETE /profile/<e>/<n> [?force=1]` | ✅ (no force) |

```console
$ isann profile use --engine llama --name highmem
[isann] active profile -> llama/highmem
        (run 'isann docker restart llama' to apply)
```

## rv

Rendezvous bookmarks (isannd owns `rvs.json`). All leaves support `--nodes`; session required.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann rv add --alias n --url <https://host:port> [--force]` | Register `alias → URL`. | `POST /rv/add` | ✅ |
| `isann rv list` | Registered RVs (`*` = default). | `GET /rv/list` | ✅ |
| `isann rv rm --alias n` | Remove by alias. | `DELETE /rv/<alias>` | ✅ |
| `isann rv use --alias n \| --clear` | Set the default RV (or clear). | `POST /rv/use` | ✅ |

```console
$ isann rv add --alias office --url https://rv.example:9000
[isann] alias "office" -> https://rv.example:9000
$ isann rv use --alias office
[isann] default RV -> "office"
```

---

## Appendix — cross-node (`--nodes`) support

| Supported ✅ | Not supported — |
|---|---|
| auth (transfer/add/rm/list) · docker (status/warmup/shutdown/ps/start/stop/restart) · list (models/loras/vaes/profiles) · model (pull/rm/list) · profile (all) · rv (all) · infer (all) · info | account (all) · favorite (all) · ghcr (all) · docker (create/rm/inspect/probe/pull) · preset (all, local) · model (info/search/register) · version |

> Every cross-node call wires through `POST /internal/api/nodes/forward` and carries the adminPath to the peer. Query-string options (`auth --roles`, `docker --force`, `profile rm --force`) are not forwarded.
