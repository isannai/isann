# 7 · Drive your node from Claude (MCP)

> Optional. Prerequisite: a running node (`isann info` responds). For the inference tools you'll also want an engine serving — **[5 · Start the engine](5-start.md)**.

`isannd` embeds a **Model Context Protocol (MCP)** server, so an AI agent like **Claude Code** can drive your node in natural language — list nodes, run inference, start and stop engines — through the same daemon entry point (`/internal/api/mcp`) the CLI uses. No extra server, no separate login.

## Connect

**1 · Issue a token.** It's a random Bearer credential — the node keeps only its hash, never your wallet key. Printed once:

```console
$ isann mcp token
<the-token>

[isann] issued MCP token tok_3f9a — shown ONCE, copy it now.
Register with Claude Code:
  claude mcp add --transport http isann \
    http://127.0.0.1:8443/internal/api/mcp \
    --header "Authorization: Bearer <the-token>"
```

Issuing a token is an **operator** action — `isann auth unlock` first.

**2 · Register the endpoint** with Claude Code (the exact command is printed for you above):

```bash
claude mcp add --transport http isann \
  http://127.0.0.1:8443/internal/api/mcp \
  --header "Authorization: Bearer <token>"
```

> Register via an environment variable — `--header "Authorization: Bearer ${ISANN_MCP_TOKEN}"` — so rotating the token only changes the variable, with no re-registration.

## Tools

The server exposes **8 tools**, one per `isann` namespace (the action is an `enum` argument — a small, fixed tool count the model can load in full). List the live set, with each tool's kind, via `isann mcp tools`:

| Tool | Ask, e.g. |
|:--|:--|
| `node_info` | "What are this node's specs and version?" |
| `list` | "List the registered nodes / installed models." |
| `infer` | "Run this prompt on llama." |
| `docker` | "Restart the sd container / show its status." |
| `model` | "Search for / remove a model." |
| `profile` | "Switch to this profile." |
| `mesh` | "Turn the provider on / off." |
| `rv` | "Add / switch the rendezvous endpoint." |

## What's locked, what's open

Authentication is **two hops with different credentials**: Claude carries only the Bearer **token** (it has no key, so it cannot sign); `isannd` adds the ECDSA **signature** with your unlocked key.

- **Read** actions (inference, listing, model search, node info) work with the token alone.
- **Control** actions (docker, mesh, `profile use`, `model rm`, …) run **only while your operator key is warm** — that is, after you've run `isann auth unlock` in a terminal. If the key is cold, the tool returns `🔒 locked — run isann auth unlock` and Claude relays that to you. Unlock once and control tools work until the session expires.
- Your **passphrase never crosses the MCP / cloud channel** — you type it only in your own terminal.

This reuses the node's normal control/data gate (see the [overview](../README.md)), so connecting Claude does **not** widen what the node allows: external inference stays sandboxed, and operation still requires an operator signature.

> **Inference is async.** The `infer` run action submits the job and returns a `job_id` right away; `status` and `result` are **separate requests**. The agent should fetch the result when the job is likely done rather than polling in a tight loop.

## Manage tokens

| Command | Description |
|:--|:--|
| `isann mcp token [--label <text>]` | Issue a token — printed once; only its hash is stored on the node. |
| `isann mcp token list` | List issued tokens (id · label · created). |
| `isann mcp token rotate <id>` | Issue a replacement, then revoke `<id>`. |
| `isann mcp token revoke <id>` | Revoke one token immediately. |
| `isann mcp tools` | Show the tools the server currently exposes (with read/control kind). |

Full flags: **[`isann` reference → mcp](../cli/cli-reference.md)**.
