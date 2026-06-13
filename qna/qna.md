# Q&A — nodes, firewall & Docker Desktop

> Troubleshooting `--nodes` (and Claude/MCP cross-node) when another node won't answer, plus Docker Desktop coexistence on Windows.

---

## Q. `isann ... --nodes <id>` to another node times out or says "peer not reachable". What do I check?

Check in this order — **the firewall is usually *not* the first cause**:

1. **Is the other node actually up, with its mesh running?**
   `isannd` has **no rendezvous (RV) connection of its own** — it borrows the control connection held by that node's **provider/broker** to look up peers. If the other node's mesh is down, peer lookup fails with **`peer not reachable` (404)** — that's *not* a firewall problem.
   - On the *target* node: `isann mesh status` → provider (and/or broker) should be **running**.
   - Start it: `isann mesh on provider --now`

2. **Is the target's `isannd` running and registered?**
   - `isann info` (node id), `isann docker status` (daemon alive).

3. **Did the target's IP / NAT change?**
   A DHCP renewal or router reboot can hand the node a new LAN IP, or change the NAT mapping. RV may still be gossiping the **old** address, so the dial lands nowhere → timeout. Restarting the target's mesh re-registers the current address.

4. **Only now: the inbound firewall** (see below). Relevant when the node is up and registered but cross-node **dials still time out** — the hole-punch reply is being dropped on the way in.

---

## Q. It worked yesterday but not today. Why?

The firewall rule that `ivm service install` adds is **program-scoped** (bound to `isannd.exe`, all profiles) and **persists** across reboots and `ivm switch`. So if cross-node worked yesterday, that rule most likely **didn't vanish** — look at what *changed* since:

| Changed since yesterday | Symptom | Fix |
|:--|:--|:--|
| Target's **mesh stopped** (logoff, reboot, manual stop) | `peer not reachable` 404 | `isann mesh on provider --now` on the target |
| Target's **LAN IP / public IP changed** (DHCP, router reboot) | dial **times out** to a stale address | restart the target's mesh so it re-registers |
| Ran `isannd` **interactively** (not as a service) and the **network profile flipped** Public↔Private | inbound dropped today | install the **program + all-profiles** rule below (survives profile changes) |
| **Antivirus / third-party firewall** updated | inbound dropped | allow `isannd.exe` inbound in *that* product (Windows rule won't cover it) |

> Rule of thumb: if it worked yesterday with the same install, check **mesh up + IP** *before* touching the firewall.

---

## Q. How do I open the Windows Firewall for `isannd`?

The hole-punch that powers `--nodes` needs **inbound** traffic to reach `isannd`. A non-interactive node (Scheduled Task / S4U) never gets the one-time Windows "allow?" popup, so the inbound reply is silently dropped unless a rule exists. The rule is **program-scoped** — it follows `isannd.exe` regardless of which ports it uses.

### Easiest — reinstall the service (it adds the rule for you)

```cmd
ivm service uninstall
ivm service install      :: prints "firewall: inbound allowed for isannd"
```

If `install` prints a `⚠ firewall ... FAILED` line, add it by hand with the script below.

### Script — `open-isannd-firewall.ps1`

Save and run it (it self-elevates to Administrator and auto-detects `isannd.exe` from the registered task):

```powershell
# open-isannd-firewall.ps1 — allow isannd inbound (cross-node hole-punch).
$ErrorActionPreference = 'Stop'
$Rule = 'isannd inbound (iSANN)'

# self-elevate to Administrator
$me = [Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()
if (-not $me.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
  Start-Process powershell "-NoProfile -ExecutionPolicy Bypass -File `"$PSCommandPath`"" -Verb RunAs
  return
}

# locate isannd.exe — from the registered Scheduled Task, else edit the fallback path
$exe = (Get-ScheduledTask -TaskName 'isannd' -ErrorAction SilentlyContinue).Actions.Execute
if (-not $exe) { $exe = 'D:\iann\bin\isannd.exe' }   # <-- set to your install root\bin\isannd.exe
if (-not (Test-Path $exe)) { throw "isannd.exe not found: $exe  — set the path manually" }

# idempotent: drop any old rule, then add program-scoped inbound allow (all profiles)
Remove-NetFirewallRule -DisplayName $Rule -ErrorAction SilentlyContinue
New-NetFirewallRule -DisplayName $Rule -Direction Inbound -Action Allow -Program $exe -Profile Any | Out-Null

Write-Host "OK - inbound allowed for $exe"
Get-NetFirewallRule -DisplayName $Rule | Format-List DisplayName, Enabled, Direction, Action, Profile
```

### One-liner — paste into an **Administrator** PowerShell

```powershell
$e=(Get-ScheduledTask isannd -EA 0).Actions.Execute; if(-not $e){$e='D:\iann\bin\isannd.exe'}; Remove-NetFirewallRule -DisplayName 'isannd inbound (iSANN)' -EA 0; New-NetFirewallRule -DisplayName 'isannd inbound (iSANN)' -Direction Inbound -Action Allow -Program $e -Profile Any
```

### Verify / remove

```powershell
# verify
Get-NetFirewallRule -DisplayName 'isannd inbound (iSANN)' | Format-List DisplayName, Enabled, Direction, Action, Profile
# remove
Remove-NetFirewallRule -DisplayName 'isannd inbound (iSANN)'
```

> **Why program-scoped, not a port?** The rule allows the `isannd.exe` binary inbound on *any* port and *every* profile (Domain/Private/Public). `ivm switch` swaps the binary at the same `…\bin\isannd.exe` path in place, so one rule stays valid across version changes — no per-port bookkeeping, and no breakage when Windows re-classifies the network.

---

## Q. Linux?

A node behind a host firewall needs `isannd`'s inbound allowed. Open it by program if your firewall supports it, or allow the listener port range. For a node that *hosts* rendezvous/gate, the server ports are:

```bash
# rendezvous + gate host (server role only)
sudo ufw allow 9000/udp      # rendezvous
sudo ufw allow 8800/tcp      # gate
sudo ufw reload
```

A plain peer node (no RV/gate) only needs `isannd` reachable inbound for hole-punch; most Linux desktops don't block outbound/established and need no change.

---

## Docker Desktop (Windows)

> iSANN uses the **native docker-ce inside WSL**, never Docker Desktop — not as a backend, not as a fallback. Full guide: **[Troubleshooting → Docker Desktop](../troubleshooting/docker-desktop.md)**.

### Q. What does "WSL Integration → Ubuntu toggle OFF" actually do?

Docker Desktop runs its engine in its own hidden WSL distro (`docker-desktop`). The **WSL Integration** toggle, *per distro*, makes Docker Desktop reach **into that distro** — it drops a `docker` CLI shim on the `PATH` and points the docker context / `/var/run/docker.sock` at **Docker Desktop's** engine. So with it **on for your Ubuntu**, typing `docker …` inside Ubuntu talks to Docker Desktop, not a native dockerd.

iSANN installs and drives its **own** native docker-ce in that same Ubuntu (`ivm setup`). **Turning the toggle off** (or quitting Docker Desktop) stops Docker Desktop from injecting its shim/socket, leaving the native dockerd to own `/var/run/docker.sock`.

- iSANN already bypasses the shim at runtime (absolute `/usr/bin/docker` + explicit `-H unix:///var/run/docker.sock`), so **off is not strictly required** — it's the clean state.
- It becomes **mandatory** if `ivm check` reports `/var/run/docker.sock is served by Docker Desktop` (Docker Desktop actually grabbed the socket → native dockerd unreachable).
- Quitting Docker Desktop has the same effect as off (nothing to inject while it isn't running).

### Q. `ivm setup` says "Docker Desktop is running." What do I do?

Quit Docker Desktop, then re-run `ivm setup`. iSANN installs and uses native docker-ce inside WSL; installing while Docker Desktop's **WSL integration** is active causes PATH/socket conflicts, so setup refuses by design.

- **Taskbar tray → Docker icon → Quit Docker Desktop**, then `ivm setup` again.
- Keeping Docker Desktop *installed but quit* is fine.

### Q. Can I keep Docker Desktop installed alongside iSANN?

Yes — they live in different WSL distros (`docker-desktop` vs your `Ubuntu`) with separate sockets. The one trap is Docker Desktop's per-distro **WSL Integration**: if it's enabled for your Ubuntu distro it injects a `docker` shim + context there. iSANN bypasses that at runtime (absolute `/usr/bin/docker` + `-H unix:///var/run/docker.sock`), but for a clean setup:

- **Docker Desktop → Settings → Resources → WSL Integration → turn the Ubuntu distro toggle off → Apply & Restart.**

`ivm check` reports Docker Desktop's state so you can confirm there's no conflict.

### Q. `ivm check` says "/var/run/docker.sock is served by Docker Desktop, not native docker-ce." 

Docker Desktop has claimed the docker socket in your Ubuntu distro, so the native dockerd isn't reachable there (the `Docker Engine` row shows **not ready**). Fix:

1. **Quit Docker Desktop**, or disable its **WSL integration** for that distro (above).
2. `isann docker warmup` — starts native dockerd inside WSL.
3. `ivm check` — `Docker Engine [OK] ... (WSL)` confirms native docker-ce.

> This guard is deliberate: rather than silently driving Docker Desktop, iSANN refuses the socket when it isn't native dockerd and tells you to clean it up.

### Q. Does Docker Desktop affect `isann model pull` / model downloads?

**No.** `isann model pull` downloads model **weights to disk** via the fetcher — it never uses docker. Docker Desktop only matters for actual docker operations (engine image `docker pull`, `docker create`, `docker ps`), which iSANN always points at native WSL dockerd.

---

## Q. How do I tell it apart — firewall vs NAT vs node down?

| You see | Most likely | Do |
|:--|:--|:--|
| `peer not reachable` **404** | target mesh **down** (RV piggyback gone) | start mesh on the target |
| dial **times out** (no 404) | inbound **firewall** *or* **stale IP / NAT** | open the firewall (above); if still failing, restart target mesh to re-register the current address |
| works on **LAN**, fails across the **internet** | **NAT / CGNAT** can't be punched | covered by relay in a later phase; for now both nodes need punchable NAT |

---

← Back to **[README](../README.md)** · **[Docker Desktop](../troubleshooting/docker-desktop.md)** · **[Port policy](../reference/ports.md)** · **[`isann` reference](../cli/cli-reference.md)**
