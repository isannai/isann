# ivm Reference (iSANN Version Manager)

[← Back to README](../README.md) · [isann reference](cli-reference.md)

> **ivm** = node lifecycle manager — a standalone binary at the install root, **independent of isannd** (so it can recover/roll back even if isannd is broken; it makes no HTTP calls). An operator only needs the `ivm` binary to start setting up a node.

## Legend

- **API**: ivm rarely touches isannd HTTP — this column says what ivm acts on.
  - `local` = local files/cert/PATH · `GitHub` = GitHub Releases · `service mgr` = OS service manager (Win Scheduled Task / Linux systemd)
- **Privilege**: `(UAC/sudo)` = elevation required. Windows **self-elevates** (UAC popup → re-runs the same command in a new admin window); Linux uses `sudo`.

## Common mechanics

1. **Root resolution** — ivm finds the install root by walking up from the `ivm` executable's location (not the cwd). So it targets the same node from any directory.
   - ⚠️ On Windows, a bare `ivm` may resolve to a different `ivm.exe` on PATH — use `.\ivm` to target a specific node.
2. **Version store layout** (nvm-style — multiple versions coexist):
   ```
   <root>/
     bin/{isann,isannd,isann-fetcher,proxy}.exe   <- active binaries (swapped by `switch`)
     conf/ · engines/ · manifests/                 <- operator state (seeded once)
     .isann/
       versions/<tag>/{bin,conf,engines,manifests,release.json}   <- cache (per-version originals)
       active                                       <- active tag pointer
       downloads/<tag>/                             <- download staging
   ```
3. **bin-only swap + seed-once** — `switch`/`use` replace only `bin/` every time. `conf/` · `engines/` · `manifests/` are seeded **once** and then preserved (operators edit conf values and engine `.env` files). A new version's changes remain in `versions/<tag>/` as a template for manual diff.
   - To reset conf to a version's defaults: empty `conf/` entirely, then `switch` again (empty → re-seed).
4. **Integrity** — `install` records `release.json` (per-file sha256 + provenance: tag/digest/url/size/time). `switch` runs `verifyVersionCache` before activating, so a corrupted cache is never activated.
5. **Privilege escalation** — `service` (install/uninstall/start/stop) and `use` control the OS service, so they need admin. On Windows, when not elevated, ivm pops a UAC window and re-runs the same command (closes on success, stays open on failure). `ivm service status` is read-only — no elevation.
6. **Bootstrap order**:
   ```
   ivm init -> ivm check -> ivm setup -> ivm install -> (ivm service install) -> ivm use/switch
   ```

---

## Index

| Command | Description | API |
|---|---|---|
| `ivm init` | First-time node setup: TLS cert + anchor + layout + PATH + activate. | `local` |
| `ivm check` | Detect OS prerequisites (WSL2 / Docker / toolkit / driver). Read-only. | `local` |
| `ivm setup` | Install OS prerequisites (Win UAC → WSL2/Docker; Linux sudo → Docker/toolkit). | `local` |
| `ivm setup drivers` | Show NVIDIA driver status + manual install guidance (no auto-install). | `local` |
| `ivm install` | Fetch a release suite into the cache (default latest). First install auto-activates. | `GitHub` |
| `ivm switch` | Raw-activate a cached version into `bin/` (service-independent). | `local` |
| `ivm service <sub>` | Register/control isannd as a service: `install` / `uninstall` / `status` / `start` / `stop`. | `service mgr` |
| `ivm use` | service stop → switch → service start (same as `switch` if no service). | `local + service mgr` |
| `ivm list` | Local cache (`-local`) or remote GitHub releases (`-remote`). | `local / GitHub` |
| `ivm rm` | Remove a cached version (active protected). Alias: `remove`. | `local` |
| `ivm version` / `--version` / `-v` | Print ivm version. | `local` |

### `ivm init`
First-time bootstrap — idempotent (preserve if present, create if absent). The only real content is the cert + anchor; everything else is empty dirs + PATH registration + an activate script.
- **Syntax**: `ivm init [--root <path>] [-cert] [-nopath]` (`-cert` = force re-issue the cert = changes node identity; `-nopath` = skip PATH registration)
- **API**: `local` — cert via `installer.EnsureCert`; PATH via shell rc (Unix) / `HKCU\Environment\Path` + `WM_SETTINGCHANGE` (Win).
- Activate the new PATH in the current shell with `call activate` (Windows) / `source ./activate` (Unix), or open a new terminal.

### `ivm check`
Read-only OS prerequisite detection — installs/creates nothing and does not wake WSL. Prints a status table + whether setup scripts are bundled + an exit code.
- **Syntax**: `ivm check [-json]`
- **API**: `local` — probes WSL/Docker/toolkit/driver locally.
- **Exit**: `0` ready / `1` missing prerequisites / `2` usage error
```console
$ ivm check
  iSANN prerequisite check  (os: windows)
    [OK]  WSL2                     installed
    [OK]  Docker Engine            reachable
    [!!]  nvidia-container-toolkit not installed
    [ -]  NVIDIA driver            driver 555 (CUDA 12.5)
  Setup scripts:  found (scripts\windows\install-isann-node.ps1)

  [!!] Not ready - missing: nvidia-container-toolkit  ->  run 'ivm setup' (UAC/sudo)
```

### `ivm setup`
Install OS prerequisites + elevate. ivm is a thin launcher — it gates (script present + DetectPrereqs) then triggers elevation; the bundled script handles idempotency / reboot / distro discovery. GPU required.
- **Syntax**: `ivm setup [-dry-run] [-force]` (`-dry-run` = preview, no elevation; `-force` = bypass already-ready)
- **API**: `local` — Win: `ShellExecute "runas"` → admin PowerShell runs `install-isann-node.ps1`. Linux: `sudo ENGINES=none bash install-isann-node.sh`.
```console
$ ivm setup
  [OK] Elevated setup launched - a new Administrator window opened.
       After it finishes (and any reboot), run 'ivm check' to verify.
```

### `ivm setup drivers`
Shows NVIDIA driver status + manual install guidance only (no privilege). ivm does not auto-install drivers (hardware/version variety + brick risk).
- **Syntax**: `ivm setup drivers`
- **API**: `local`

### `ivm install`
Fetch a release zip into the local cache (`.isann/versions/<tag>/`). Default latest; `--version` for a specific tag. Range-resume + digest verify + per-file hash → `release.json`. Skips if already installed (release.json digest matches). **First install (no active version) auto-runs `switch`.**
- **Syntax**: `ivm install [--version <tag>] [--token <gh-token>]`
- **API**: `GitHub` — `GET /repos/isannai/isann/releases/{latest,tags/<tag>}` → asset `isann-<os>-<arch>.zip` fetch → digest verify → ExtractZip → staging → atomic rename.
- **⚠️ Tags match exactly** — `--version` must equal the GitHub release tag verbatim. If the page shows `0.1.2`, use `--version 0.1.2`; if `v0.1.1`, use `--version v0.1.1`. ivm does not add or strip a `v` prefix — **copy the tag from the Releases page**.
```console
$ ivm install --version 0.1.2
0.1.2  downloading isann-windows-amd64.zip  43.6 MB
  [==============================]  100%
0.1.2  extracted -> .isann/versions/0.1.2 (bin conf engines manifests)
0.1.2  release.json written (digest verified)
Switched to 0.1.2 -- bin/ replaced (version binaries).   # first install only
```

### `ivm switch`
Raw-swap the cached `<tag>` into the live root — **does not touch the service** (a manual escape-hatch that works even when the service is stopped/absent). Replaces `bin/` + seeds `conf/engines/manifests` once + updates `.isann/active`. Runs `verifyVersionCache` (release.json check) before activating.
- **Syntax**: `ivm switch --version <tag>`
- **API**: `local` — no privilege. A running isannd.exe is locked on Windows, so it is moved aside via the `.exe.old` trick (the live daemon keeps the old binary until restarted).
```console
$ ivm switch --version 0.1.0
Switched to 0.1.0 -- bin/ replaced (version binaries).
  preserved (operator state, not overwritten): conf/ engines/ manifests/
```
> If the service is up during `switch`, `bin/` is replaced but the running daemon stays on the old binary — restart it (`ivm service stop` → `ivm service start`) or use `ivm use`.

### `ivm service`
Register/control isannd as a service/daemon. It targets the stable path `<root>/bin/isannd[.exe] -config <root>/conf/isannd.json`, so a version swap needs no re-registration. **Registration is only via this command** (`use` does not register).
- **Syntax**: `ivm service <install | uninstall | status | start | stop>`
- **API**: `service mgr` — **Windows = on-demand Scheduled Task (LogonType S4U)** (see Appendix A) / **Linux = systemd** (`register-isannd.sh` + `systemctl`).
- **Privilege**: install/uninstall/start/stop = UAC (Win, self-elevate) / sudo (Linux). **`status` is read-only** (no privilege).

| sub | action |
|---|---|
| `install` | Register the task/unit (no auto-start — on-demand). **Windows** also adds a program-scoped inbound firewall rule for `isannd.exe` (required for cross-node hole-punch under the non-interactive S4U task). Start it separately with `start`. |
| `uninstall` | Unregister (+ remove the firewall rule on Windows). |
| `status` | Show state (Ready / Running / not installed) — no privilege. |
| `start` | Start. |
| `stop` | Stop. |

```console
$ .\ivm service install      # UAC -> S4U Scheduled Task (no stored password)
Task "isannd" installed -- S4U, runs as MACHINE\user, survives logoff.
  firewall: inbound allowed for isannd (cross-node hole-punch)
$ .\ivm service start
$ .\ivm service status       # no privilege needed
Task "isannd": Running  (lastRun: ..., lastResult: 267009)
```
> `lastResult: 267009` (= `0x41301 SCHED_S_TASK_RUNNING`, "task currently running") is normal, not an error.
>
> **Windows firewall (cross-node)**: a non-interactive S4U task never gets the interactive Firewall "allow?" prompt a console run would, so without an inbound rule the peer's hole-punch / dial replies are dropped and `--nodes` dials time out. `install` adds a program-scoped inbound rule for `isannd.exe`; `uninstall` removes it — operators don't touch the firewall.

### `ivm use`
`switch` wrapped in service control. If a service is registered: **service stop → switch → service start** (ends running on the new version). If not registered: same as raw `switch` + a hint (no privilege). **Does not register.**
- **Syntax**: `ivm use --version <tag>`
- **API**: `local + service mgr` — registration is detected with a no-admin query; if registered, stop/start need privilege → the whole `use` self-elevates (Win) / per-op sudo (Linux). On a switch failure it best-effort restarts to avoid leaving the node down.
```console
$ .\ivm use --version 0.1.2
Requesting Administrator privileges (UAC)...
# (admin window) Stopping -> Switching to 0.1.2 -> Starting -> [OK] use complete
$ bin\isann version
isann CLI  0.1.2
isannd     0.1.2 (running at http://127.0.0.1:8443)
```

### `ivm list`
Local cache versions (default / `-local`) or remote GitHub tags (`-remote`). Both flags at once = error.
- **Syntax**: `ivm list [-local | -remote [--pf <os,..>] [--limit N] [--token <t>]]`
- **API**: `-local` → `local` (scan `.isann/versions/` + active `*`) / `-remote` → `GitHub` (`GET /repos/isannai/isann/releases`, anon 60/h). `--pf` = platforms shown in the PLATFORMS column (comma; default = this host); `--limit N` = row cap (0 = all).
```console
$ ivm list
TAG     STATUS
0.1.2   * active
0.1.0     installed
```

### `ivm rm` (alias `remove`)
Remove one version from the cache. **The active tag is protected** (error — switch to another first). Network: none.
- **Syntax**: `ivm rm --version <tag>` (or `ivm remove --version <tag>`)
- **API**: `local` — `RemoveAll` on `versions/<tag>/`; active protection compares `.isann/active`.
```console
$ ivm rm --version 0.1.0
Removed 0.1.0 from cache.
```

### `ivm version`
Print the ivm version (`setup.IvmVersion`, injected at build via ldflags — see Appendix B).
- **Syntax**: `ivm version` | `ivm --version` | `ivm -v`
- **API**: `local`

---

## Appendix A — Windows service model (S4U Scheduled Task)

On Windows `ivm service` registers isannd as an **on-demand Scheduled Task**, not an SCM service. Derived from four operator requirements:

| # | Requirement | Met by |
|---|---|---|
| 1 | isannd survives logoff | "run whether logged on or not" = **Session 0** |
| 2 | Manual start/stop | no trigger + `service start`/`stop` |
| 3 | isannd reaches **per-user WSL/Docker** (`docker warmup`) | runs as **your account** (sees your HKCU distro) |
| 4 | No boot/login auto-start (WSL+GPU are heavy) | no trigger |

**Why not SCM**: the default LocalSystem account **cannot see per-user WSL distros** (`wsl -l -v` → `exit 0xffffffff`). A user-account SCM service would need `SeServiceLogonRight` granted via LSA (not exposed by `golang.org/x/sys/windows`, fragile) plus a stored password.

**S4U (Service-for-User)** is used: Task Scheduler grants the logon right itself, and **stores no password** (it mints a user identity token via system privilege) → **immune to Windows password changes**. The token is local-only (no network credentials), which is sufficient for local WSL/Docker.

- isannd runs as a **plain process** under the Task (`svc.IsWindowsService()` = false → console path; Session 0 → no window). `stop` terminates the process (the fetcher is reaped by the OS Job Object).
- **Migrating from an old SCM service**: delete it with `sc delete isannd` (admin), then `ivm service install` to re-register as a Task.

## Appendix B — Building a version (build/windows/build.bat)

Release zips are produced by `build/windows/build.bat`. **A version argument is required** (without it the build fails).
- **Usage**: `build\windows\build.bat <version>` (e.g. `build\windows\build.bat 0.1.2`)
- Outputs (`build\windows\out\`):
  - `isann-windows-amd64.zip` — node suite (what `ivm install` fetches): `bin/{isann,isannd,isann-fetcher,proxy}+*.bat` + `conf/` + `engines/` + `manifests/`
  - `ivm-windows-amd64.zip` — bootstrap: `ivm.exe` + `scripts/{windows,linux}/`
- The version is injected via **ldflags `-X`** into `pkg/setup.*Version`.
- Upload `isann-windows-amd64.zip` to a GitHub release; `ivm install --version <tag>` fetches it. GitHub auto-computes the asset digest → ivm verifies it.
- A Linux equivalent (`build/linux/build.sh`) is still to come.
