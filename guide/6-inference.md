# 6 · Run inference

> Prerequisite: an engine that's up and serving — **[5 · Start the engine](5-start.md)** (`isann docker probe <engine>` ready).

`isann infer` submits jobs through your local `isannd`, so sessions, output shaping, and cross-node routing all work the same way.

## Local

Inference is **async** — `run` queues the job on the provider and returns a `job_id` immediately, so a long render never holds the connection open. Fetch the result when it's ready:

```console
$ isann infer run --engine llama --prompt "Summarize in one line: ..."
job_id: j-7f3a    service: llm-api    queue: 0/8

$ isann infer status j-7f3a        # optional — poll progress
status: running   (45%)

$ isann infer result j-7f3a
The summarized text appears here.
```

- `--engine` accepts an engine name (`llama`), a service name (`llm-api`), or a modality (`text`).
- For image engines, save the result to a file: `isann infer result <job_id> --out out.png`.

## Local — chat

```cmd
isann infer chat --engine llama --system "You are a helpful assistant."
```

A multi-turn REPL (`/reset`, `/exit`).

## Cross-node — run on another node

Point a job at a peer node with `--nodes <nodeID>` (a `P:0x…` provider id, or a saved favorite alias). Your `isannd` reaches the peer over P2P directly — **no broker needed on your side**:

```console
$ isann infer run --engine llama --prompt "안녕" --nodes P:0x70bb…d5da
job_id: a5d7eb6599c8    service: llm-api    queue: 1/50

$ isann infer result a5d7eb6599c8 --nodes P:0x70bb…d5da
안녕하세요! 어떻게 도와드릴까요?
```

- The rendezvous (RV) used for discovery is your `isann rv use` default; override per-command with **`--rv <alias|url>`**.
- **Apply `--nodes` (and `--rv`) to every sub-command for a job** — the job lives on the remote node, so `isann infer result <job_id> --nodes P:0x…` is required to fetch it. Without `--nodes`, `status`/`result` query your local provider and return 502.

> Set up RVs first: `isann rv add --alias main --url https://<rv-host>:9000`, then `isann rv use --alias main`. See **[`isann` reference → rv](../cli/cli-reference.md)** (`--control` overrides the derived control port for a non-standard RV).

## Sub-commands

| Command | Description |
|:--|:--|
| `isann infer run --engine e [--<param>…] [--nodes id] [--rv r]` | Submit a job — returns a `job_id` (async). |
| `isann infer chat --engine e [--system s] [--temperature f] [--max-tokens n]` | Multi-turn REPL. |
| `isann infer status <id> [--nodes id]` | Job status. |
| `isann infer result <id> [--out f] [-consume] [--nodes id]` | Fetch the result. |
| `isann infer rm <id> [--nodes id]` | Delete a finished job. |
| `isann infer queue --engine e [--nodes id]` | Service queue stats. |

Extract a single field from a result with `--proj`, e.g. `--proj choices[0].message.content`. Reuse parameter sets with `isann preset`.

Full flags: **[`isann` reference → infer](../cli/cli-reference.md)**.
