# 4 · Profiles (optional)

> Prerequisite: an engine you can configure — see **[3 · Configure the engine](3-config.md)**.

A **profile** is a named snapshot of an engine's `.env`. Instead of hand-editing `.env` every time, save several configs and switch between them — e.g. a `lowmem` profile (few GPU layers) and a `highctx` profile (large context). `isannd` owns the profile store; a session is required.

## Workflow

```cmd
isann profile create --engine llama --name highctx    :: new profile (prompts for values)
isann profile list --engine llama                      :: see profiles (* = active)
isann profile use --engine llama --name highctx        :: make it active (copies to .env)
isann docker restart llama                              :: apply
```

`profile use` copies the profile into the engine's `.env`, then you **restart** the engine to apply — same as editing `.env` directly, but reversible and named.

## Commands

| Command | Description |
|:--|:--|
| `isann profile list [--engine e]` | List profiles (`*` = active). |
| `isann profile show --engine e --name n` | Print a profile's `.env`. |
| `isann profile create --engine e --name n [-y] [--force]` | Create a new profile. |
| `isann profile update --engine e --name n` | Edit (current values as defaults). |
| `isann profile copy --engine e --from a --to b` | Clone a profile. |
| `isann profile use --engine e --name n` | Set active (copies to `.env`). |
| `isann profile rm --engine e --name n [-y] [--force]` | Delete (active needs `--force`). |

```console
$ isann profile use --engine llama --name highctx
[isann] active profile -> llama/highctx
        (run 'isann docker restart llama' to apply)
```

All profile leaves support `--nodes` for cross-node management.

## Next

→ **[5 · Start the engine](5-start.md)**.
