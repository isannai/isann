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

## Common flags

Shared across commands — listed once here instead of repeating per command. Each namespace's table below still carries a per-command **Nodes** column for detail.

| Flag(s) | Applies to |
|:--|:--|
| **`-json`** · **`-pretty`** · **`--proj <p>`** — output shaping (raw / indent / field projection) | **all read commands** (every namespace) |
| **`--nodes <id>`** — run on a peer node (cross-node) | **✅** `info` · `infer` · `mesh` · `profile` · `rv` (all sub-commands) · `auth` (transfer/add/rm/list) · `docker` (status/warmup/shutdown/ps/start/stop/restart) · `list` (models/loras/vaes/profiles) · `model` (pull/rm/list)  —  **✗** `account` · `favorite` · `ghcr` · `mcp` · `preset` |

> Cross-node wires through `POST /internal/api/nodes/forward` (adminPath in body). **Query-string** options are not forwarded (`auth --roles`, `docker --force`, `profile rm --force`); body options are.

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

Pure local — calls `pkg/wallet` directly, never touches isannd (except `whoami`, which reads the daemon's unlock session). `--nodes` not supported.

| Command | Description | API |
|---|---|---|
| `isann account create --alias n [--force]` | New keystore + register alias → address. Passphrase ≥ 8 chars. | `local` |
| `isann account import <path> --alias n` · `--pk <hex> --alias n` | Hardlink an existing keystore, or encrypt a raw 0x-hex private key. | `local` |
| `isann account list [-json]` | Alias / address / file / owner-role table. | `local` |
| `isann account whoami [-json]` | Print the alias **currently unlocked** in the isannd session (reverse-mapped from accounts.json) + its **session expiry and remaining TTL**; `not unlocked` otherwise. | `GET /auth/status` + local |
| `isann account pk --alias a` | Decrypt + print the raw private key (passphrase prompt + warning). | `local` |
| `isann account rm --alias a [-y]` | Remove a keystore (owner key protected — transfer first). | `local` |

```console
$ isann account create --alias office
[isann] generated 0xab12...cd34
[isann] alias "office" -> 0xab12...cd34

$ isann account whoami
office
  expires  2026-06-10T15:42:18+09:00
  ttl      38m12s
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

## conn

Connectivity diagnostics between nodes. Session required (the probe is a signed cross-node round-trip).

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann conn ping --nodes <list> [--count N]` | Dial each node through the real cross-node path (RV lookup + hole-punch + HTTP/3) and report reachability + round-trip latency + the address it connected through. | `GET /conn/ping?node=<n>` | ✅ (fan-out) |

```console
$ isann conn ping --nodes office,lab,0xBADa…7CEe
NODE      RESULT   RTT       VIA
office    ok        42.1 ms  203.0.113.7:7443 (wan)
lab       ok         7.8 ms  192.168.0.5:7443 (lan)
0xBADa…   timeout      —     —
```

`--nodes` takes one target or a comma list; favorites (aliases) resolve like every other cross-node command. `--count N` (1-10) pings each node N times and reports the **best** RTT. **VIA** is the address that answered — `lan` (private IP) vs `wan` (public; direct or hole-punched, not yet distinguished). Reachability = *the peer answered* — even a `403` (you're not its operator) proves the connection punched through, so it counts as `ok`. Per-node isolation: one node failing never drops the others.

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
| `isann infer run --engine e [--<param>..] [-stdin] [--out f] [--preset n] [--nodes id] [--rv alias\|url] [--provider h:p]` | Submit a job → `job_id` (async; fetch with `result`). | `POST {base}/svc/<svc>/v1/jobs` | ✅ |
| `isann infer chat --engine e [--system s] [--temperature f] [--max-tokens n]` | Multi-turn REPL (`/reset`, `/exit`). | `POST {base}/svc/<svc>/v1/jobs` per turn | ✅ |
| `isann infer status <id>` | Job status. | `GET {base}/v1/jobs/<id>` | ✅ |
| `isann infer result <id> [--out f] [-consume]` | Finished result (text→stdout, image/audio→file). | `GET {base}/v1/jobs/<id>/result` | ✅ |
| `isann infer rm <id>` | Delete a finished job. | `DELETE {base}/v1/jobs/<id>` | ✅ |
| `isann infer queue --engine e` | Service queue stats. | `GET {base}/v1/queue/stats?service=<svc>` | ✅ |

```console
$ isann infer run --engine text --prompt "Summarize in one line: ..."
job_id: j-7f3a    service: llm-api    queue: 0/8
$ isann infer result j-7f3a
The summarized text appears here.

$ isann infer result j-9b1 --out out.png
saved out.png (2451233 bytes)
```

For `result`, `--proj` extracts a field — e.g. `--proj choices[0].message.content`. `infer` shapes output client-side.

**Cross-node (`--nodes`)**: isannd self-dials the RV and dials the peer directly — **no broker needed on the client**. The RV is the `rv use` default, overridable per-command with `--rv <alias|url>` (an alias also carries that RV's stored control-addr override; a bare URL derives `host:9100`). `run`/`chat`/`status`/`result`/`rm`/`queue` all accept `--rv`. Pass `--nodes` (and `--rv`) to **every** sub-command for a job — the job lives on the remote node, so `status`/`result` without `--nodes` query the local provider and 502.

## list

Read-only views. models/loras/vaes/profiles support `--nodes`. nodes/metrics query the RV (node-agnostic). favorites/rvs/accounts/presets delegate to other namespaces.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann list nodes [--rv r] [--role x] [--owners a,b] [--model m] [--limit N] [--page N] [-no-cache]` | Nodes on an RV. The full list is fetched once and **ETag-cached** (revalidated every call — never stale, 304 when unchanged); `--role` / `--owners` / `--model` filter and `--page`/`--limit` paginate **client-side** (paging never re-fetches). | `GET /list/nodes?rv=<url>` | — |
| `isann list metrics [--rv r]` | Per-(node,service) metrics from the RV (always fresh — volatile, no cache). | `GET /list/metrics?rv=<url>` | — |
| `isann list models\|loras\|vaes [--engine e]` | Scan `engines/<e>/models/` (loras/vaes are KIND-filtered views). | `GET /list/models [?engine=<e>]` | ✅ |
| `isann list profiles [--engine e]` | Scan `engines/<e>/profiles/*.env`. | `GET /list/profiles [?engine=<e>]` | ✅ |
| `isann list favorites \| rvs \| accounts \| presets` | Delegate to favorite / rv / account / preset. | (delegated) | partial |

`list nodes` filter matching: **`--role`** exact — `provider` \| `broker` only (consumers register for hole-punch and are **discovery-hidden**, so they never appear here; their counts live in `list rvs -remote`); **`--owners`** lowercase **prefix** (comma list, OR) — not exact; **`--model`** case-insensitive **substring** of a served model name — not exact. `-no-cache` forces a fresh fetch.

```console
$ isann list nodes --rv office --role provider --model qwen --limit 10
ID          ROLE      ADDR          OWNER     MODEL        TPM
p:0xabc...  provider  1.2.3.4:7443  0xabc...  qwen2.5:14b  verified
[isann] 1 node(s)

$ isann list models --engine llama
ENGINE  KIND   TYPE    REG  NAME                 SIZE
llama   model  single   v   Qwen2.5-1.5B-Q4_K_M  1.0 GiB
```

## mcp

Expose this node to an **MCP client** (Claude Code, etc.) so it can drive the node's tools in natural language. isannd embeds an MCP server (Streamable HTTP, JSON-RPC 2.0) at **`POST /internal/api/mcp`**; `isann mcp` manages the Bearer tokens a client uses to reach it. `--nodes` not supported (token management is per-node local).

> The token is the **client→isannd** credential — a random opaque secret (stored as a **hash** in `artifacts/mcp.json`, raw shown once), **not** a signature or wallet key. Signing stays an isannd-side keycache concern.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann mcp tools` | List the tools the MCP server exposes. | `GET /mcp/tools` | — |
| `isann mcp token [--label t]` | Issue a token (printed **once** — copy it now). | `POST /mcp/tokens` | — |
| `isann mcp token list` | List issued tokens (id + label + created). | `GET /mcp/tokens` | — |
| `isann mcp token revoke <id>` | Revoke a token by id. | `DELETE /mcp/tokens/{id}` | — |
| `isann mcp token rotate <id>` | Issue a new token and revoke `<id>`. | (issue + delete) | — |

```console
$ isann auth unlock              # issuing a token is an operator action
$ isann mcp token
b1c2d3e4f5...9e0f
[isann] issued MCP token a1b2c3d4e5f6 — shown ONCE, copy it now.

$ claude mcp add --transport http isann \
    http://127.0.0.1:8443/internal/api/mcp \
    --header "Authorization: Bearer b1c2d3e4f5...9e0f"
```

**Tools exposed** — consolidated one-per-namespace (keeps the model's tool context small; operations are an `action`/`what` enum, not separate tools):

| Tool | Kind | Args |
|---|---|---|
| `node_info` | read | — |
| `list` | read | `what`: nodes \| models \| profiles \| containers \| mesh \| rvs |
| `conn` | control* | `action`: ping · `node` (id\|alias) — *cross-node* reachability + latency; needs unlock (signs) |
| `infer` | read | `action`: schema \| run \| status \| result · `engine` · `input{}` · `job_id` |
| `docker` | control | `action`: start \| stop \| restart \| rm · `name` · `force?` |
| `mesh` | control | `action`: start \| stop \| on \| off · `component`: provider \| broker · `now?` |
| `profile` | control | `action`: get \| use · `engine` · `name` |
| `rv` | control | `action`: add \| rm \| use · `alias` · `url?` |
| `model` | mixed | `action`: info \| search \| rm · (`url`/`query`/`engine`+`kind`+`name`) |

`isann mcp tools` prints this list. **Control actions** require an **unlocked operator** — when the node is locked they return a *"locked — run `isann auth unlock`"* result so the assistant can prompt you (read actions like `infer`, `list`, `model search` are not gated). Gating is **per action**, so a tool's reads work while locked even if its writes don't. Tools act on **this node** — except `conn`, which probes another node via a signed cross-node round-trip; `--nodes`-style cross-node targeting for the other tools is a later addition. e.g. *"stop llama"* → `docker(action=stop, name=llama)`; *"generate an image"* → `infer(action=run, engine=sd, input={…})`.

> Claude Code connects over HTTP directly. stdio-only clients bridge via `mcp-remote`. Endpoint is loopback-only (`127.0.0.1`) with an Origin guard.

## mesh

Host-native backends — `provider` and `broker`. They're the lightweight `proxy` binary (**not** containers: they couple to isannd over loopback, so they live on the host), and **isannd spawns them as managed child processes** (procguard — they die with isannd, and `ivm use` restarts them on the new binary). **opt-in**: a node with none enabled runs isannd alone — a client-only node still does cross-node inference via self-dial. Operator-only; session required. `--nodes` targets a **peer's** backends (your operator role on that node is enforced).

> **on/off = autostart** (persisted in `artifacts/mesh.json`, like `systemctl enable/disable`) · **start/stop = right now** (runtime, like `systemctl start/stop`). Independent.

| Command | Description | API | Nodes |
|---|---|---|---|
| `isann mesh status` | provider/broker autostart + runtime state (PID, listen addr). | `GET /mesh/status` | ✅ |
| `isann mesh start <provider\|broker>` | Spawn now (runtime only — not persisted). | `POST /mesh/start` | ✅ |
| `isann mesh stop <provider\|broker>` | Kill the running process now. | `POST /mesh/stop` | ✅ |
| `isann mesh on <provider\|broker> [--now]` | Enable autostart (saved to `artifacts/mesh.json`). `--now` also starts it immediately. | `POST /mesh/on` | ✅ |
| `isann mesh off <provider\|broker> [--now]` | Disable autostart. `--now` also stops it now. | `POST /mesh/off` | ✅ |

```console
$ isann mesh on provider --now
[isann] provider: autostart on, started (pid 12345)

$ isann mesh status
COMPONENT  AUTOSTART  STATE     PID     LISTEN
provider   on         running   12345   :8090
broker     off        stopped   -       :8080
```

**provider** serves local inference (needed for a node that serves its own GPU); **broker** is the optional management UI. Many nodes need neither — `isann infer --nodes` reaches other nodes' providers directly.

**Cross-node (`--nodes`)**: manage a **peer's** backends — `isann mesh on provider --nodes P:0x…` enables + starts the provider on a remote node you operate (gated by your operator role there). `isann mesh status --nodes a,b,c` fans in a NODE-column table. Same mechanism as `docker start --nodes` (remote process control).

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
| `isann rv add --alias n --url <https://host:port> [--control h:p] [--force]` | Register `alias → URL`. Control addr is derived `host:9100`; `--control` overrides for a non-standard RV. | `POST /rv/add` | ✅ |
| `isann rv list [-remote] [-no-cache]` | Local: registered RVs — `URL`, `CONTROL` (derived/overridden), `*` = default. **`-remote`**: ALL RVs from Gate with per-RV `NODES / PROVIDERS / BROKERS / CONSUMERS / PUBLIC` counts (ETag-cached). | `GET /rv/list` · `-remote`: `GET /internal/gate/v1/rendezvous` | ✅ |
| `isann rv rm --alias n` | Remove by alias. | `DELETE /rv/<alias>` | ✅ |
| `isann rv use --alias n \| --clear` | Set the default RV (or clear). | `POST /rv/use` | ✅ |

```console
$ isann rv add --alias office --url https://rv.example:9000
[isann] alias "office" -> https://rv.example:9000  (control rv.example:9100)
$ isann rv add --alias custom --url https://rv2.example:9000 --control rv2.example:19100
[isann] alias "custom" -> https://rv2.example:9000  (control rv2.example:19100)
$ isann rv use --alias office
[isann] default RV -> "office"
```

The **control addr** is the RV TCP endpoint isannd self-dials for cross-node lookups. By default it's the URL host + port `9100`; `--control <host:port>` overrides it for an RV on a non-standard control port. `isann rv list` shows the effective value in the `CONTROL` column.

`-remote` counts are the Gate's live aggregation of each RV's registered nodes. **`CONSUMERS`** counts client-only nodes (role `consumer`) that register only for NAT hole-punch — they are **counted but never listed** in discovery (`list nodes` / the directory), by design.

> Cross-node support is summarized in **[Common flags](#common-flags)** at the top.
