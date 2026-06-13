# RV admission (public / protected)

> How a rendezvous controls **who may register**. Default is open; protected mode gates registration on an issuer-signed credential.

## Modes

A rendezvous reads its **own `auth.json`**, sitting next to its `rendezvous.json` (the RV is deployed separately from any node, so this file is the RV operator's, not a node's):

```json
{
  "mode": "protected",
  "issuers": [
    { "address": "0xGAME…", "bind": "node" },
    { "address": "0xOPEN…", "bind": "none" }
  ]
}
```

| `mode` | Behavior |
|:--|:--|
| `public` (default) | Anyone with a valid register signature may register. `issuers` is ignored. `open` is a legacy alias for `public`. |
| `protected` | A registering node must present a valid **issuer-signed admission credential**. Applies to **all roles** — provider, broker, and consumer. |

A missing `auth.json` ⇒ `public`. `mode: protected` with **no** issuers is a startup error (a protected RV that admits nobody is almost always a misconfiguration).

## Issuers

Each entry authorizes one **issuer** — the party (e.g. a game company, a consortium operator) whose key signs admission credentials. The issuer is identified by its **address**; the signature recovers to it, so the issuer address is never sent on the wire.

| Field | Meaning |
|:--|:--|
| `address` | Issuer wallet address (lowercased on load). |
| `bind` | What shape of credential this issuer may present (below). Omitted ⇒ `node`. |

A credential's **lifetime** is the absolute expiry the issuer signs into it (see [The credential](#the-credential)) — there is no RV-side ttl. Trust is binary: an issuer listed here is trusted to set its own expiry, and revocation = removing it from this list (+ restart).

### `bind` — node-bound vs bearer

| `bind` | Credential | Admits |
|:--|:--|:--|
| `node` (default) | Bound to a specific node EOA | **that node only** |
| `none` | Bearer (no node binding) | **any node** that presents it |
| `any` | Either form accepted | both |

- **`node`** is the safe default: a stolen credential is useless without the node's private key (the register frame also carries the node's own signature). Use for nodes you authorize individually.
- **`none`** (bearer) trades that safety for convenience — one credential admits any node that holds it (like an invite code). A leaked bearer credential lets anyone register, so pair it with a **near-term expiry** and a secure distribution channel.

The RV reconstructs both message shapes and accepts whichever recovers to an authorized issuer **whose policy permits that shape** — the signature itself prevents mixing (a bound credential never verifies as bearer, and vice versa).

## The credential

The issuer signs a canonical string with **standard `personal_sign` (eth_sign)** — so any Ethereum wallet works, no iSANN tooling required:

```
node-bound:  ISANN-CREDENTIAL:<lowercase node EOA>:<issued_ms>:<expire_ms>
bearer:      ISANN-CREDENTIAL::<issued_ms>:<expire_ms>                       (empty middle)
```

`<issued_ms>` is the issuance time and `<expire_ms>` the absolute expiry, both Unix milliseconds. The RV admits only while `now ≤ expire_ms` (and rejects a credential with `expire_ms ≤ 0`). `issued_ms` is signed for audit; it no longer gates anything.

### Issuer signs

Turnkey, with an iSANN keystore (no isannd needed) — stamps `issued_ms`, takes the required `--expire`, and prints the operator's install line:

```console
$ isann account issue --alias issuer --node 0xabc… --expire 2026-12-31   # bearer: -bearer instead of --node
  …  isann cred add --alias <name> --sig 0x… --issued 1733972400000 --expire 1798675199000
```

Or sign the canonical string directly with any wallet (compute `expire_ms` yourself):

```console
# node-bound (cast / foundry)
$ cast wallet sign --private-key 0xISSUER_PK "ISANN-CREDENTIAL:0xabc…:1733972400000:1798675199000"

# bearer
$ cast wallet sign --private-key 0xISSUER_PK "ISANN-CREDENTIAL::1733972400000:1798675199000"
```

```js
// ethers.js
const msg = `ISANN-CREDENTIAL:${nodeAddr.toLowerCase()}:${issuedMs}:${expireMs}`; // bearer: `ISANN-CREDENTIAL::${issuedMs}:${expireMs}`
const sig = await issuerWallet.signMessage(msg);                                  // → { sig, issuedMs, expireMs } to the operator
```

> ⚠️ The `issued`/`expire` passed to `isann cred add` **must equal** the values that were signed — the issuer hands over `sig`, `issued_ms`, **and** `expire_ms` together. Sign the bare EOA from `isann info` (the node_id is already the prefix-less, lowercased EOA); do **not** include the `P:`/`B:`/`C:` role prefix.

## Who issues — and who does not

| Role | Holds issuer key? |
|:--|:--|
| **Issuer** (game company / consortium authority) | **Yes** — signs credentials. Defined by holding the key listed in `auth.json`. |
| **Node** | No — a node that self-signs would defeat the gate. |
| **RV** | No — if the RV held the issuer key it would both sign and verify, which is pointless indirection (use a direct allowlist instead). Signed credentials exist precisely so the issuer can be **separate** from the RV (delegated / third-party / offline / multiple issuers). |

## Node side

The node operator adds the credential to its pool and isannd attaches the **active** one to register frames automatically. Credentials are issuer-bound (not RV-bound), so a node can keep several and switch the active one with `cred use` — no re-issuance. See **[`isann` reference → cred](../cli/cli-reference.md#cred)**:

```console
$ isann info                                         # → node_id (give to issuer)
$ isann cred add --alias game-rv --sig 0x… --issued 1733972400000 --expire 1798675199000
$ isann cred use --alias game-rv                     # (first credential is auto-active)
$ isann cred list                                    # ALIAS / ISSUER / EXPIRE / STATUS / BIND / ACTIVE
```

Stored at `artifacts/cred.json` and re-read on every FullSync register, so a swapped credential (or `cred use`) is picked up without restarting isannd.

## Related

- [`isann` reference → cred](../cli/cli-reference.md#cred) — `cred add` / `list` / `use` / `rm`
- [`isann` reference → account](../cli/cli-reference.md#account) — `account issue` (issue a credential)
- [Port policy](ports.md)
