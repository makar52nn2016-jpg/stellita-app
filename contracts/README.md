# Contracts

This directory holds the catalog of Stellar Soroban contracts that Stellita
ships — the committed WASM, the build script that produced them, the deploy
script that wires them into the testnet deploy path, and the JSON manifests that
the LLM catalog is built from. **No Rust runs in production.** The committed
WASM is what ships; everything else is here so a contributor can rebuild,
verify, and extend the catalog.

If you are here to add or change a contract, jump to
[Adding a contract type](#adding-a-contract-type) at the bottom of this file.
If you are here to rebuild the catalog, start with
[Rebuilding the WASM](#rebuilding-the-wasm). If you are here to deploy a
contract manually on testnet (without going through the Express backend), jump
to [Dry-running `deploy.mjs`](#dry-running-deploymjs).

---

## Layout

```
contracts/
├── build.sh                 # Reproducibly rebuilds every WASM in wasm/
├── scripts/
│   └── deploy.mjs           # Standalone testnet deploy (mirrors server/_lib/deploy.ts)
├── manifests/               # One JSON per contract — the LLM catalog source
│   ├── oz-ownable.json
│   ├── oz-fungible-token.json
│   ├── oz-nft.json
│   └── soroswap-router.json
└── wasm/                    # Committed .wasm binaries — this is what ships
    ├── ownable.wasm
    ├── fungible-token.wasm
    └── nft.wasm
```

`contracts/wasm/` is git-tracked. `contracts/.oz-src/` (the cloned OpenZeppelin
working tree used during the build) is gitignored — every build re-clones it
from the pinned tag.

---

## Rebuilding the WASM

`./contracts/build.sh` clones OpenZeppelin `stellar-contracts` at the pinned
tag `OZ_TAG` (currently `v0.7.2` — see the top of `build.sh`), compiles each
entry in the `CATALOG` array, and copies the resulting `.wasm` into
`contracts/wasm/`. The script prints the SHA-256 prefix and byte count for
every emitted binary so you can confirm the build matched the committed file.

### Requirements (build machine only)

These tools are needed **on the build machine only** — the server never runs
Rust. They are not listed in `package.json` because they are not part of the
runtime.

- `rust` + `cargo` with the `wasm32-unknown-unknown` target installed
- `stellar-cli` >= `25.2.0` (Soroban SDK 26 needs `experimental_spec_shaking_v2`)

A macOS/Linux machine is the simplest path. On Windows, use WSL2 — `stellar-cli`
and the Rust toolchain both work there.

### Commands

From the repository root:

```bash
# Rebuild every contract in the CATALOG array
./contracts/build.sh
```

That is the only command. It is idempotent — the script skips the `git clone`
of OpenZeppelin if `.oz-src/.git` already exists, so a re-run only re-compiles.

### Expected output

The script prints one line per catalog entry, e.g.:

```
→ Cloning OpenZeppelin stellar-contracts v0.7.2 …
→ Building ownable-example → wasm/ownable.wasm
   <sha256-prefix>  (<bytes> bytes)
→ Building fungible-pausable-example → wasm/fungible-token.wasm
   <sha256-prefix>  (<bytes> bytes)
→ Building nft-sequential-minting-example → wasm/nft.wasm
   <sha256-prefix>  (<bytes> bytes)
✅ Catalog built into contracts/wasm/
```

### Verifying against the committed WASM

The committed `.wasm` files in `contracts/wasm/` are what the server deploys.
After rebuilding, the new binary must match the committed one byte-for-byte
unless you intentionally changed the catalog. To verify:

```bash
shasum -a 256 contracts/wasm/*.wasm
```

Compare the hashes to the ones `build.sh` just printed. If they differ, either
the catalog changed (expected) or the upstream pinned tag was force-pushed
(investigate before committing).

> **Commit rebuilt WASM.** If you intentionally change `CATALOG`, the
> `OZ_TAG`, or anything else that changes the emitted bytes, you MUST commit
> the new `.wasm` files in the same PR — the serverless deploy path reads the
> committed bytes, not the build output. See
> [Adding a contract type](#adding-a-contract-type) for the full checklist.

> **CI hash check.** The repository's quality gate runs
> `pnpm typecheck && pnpm lint && pnpm build` before any PR can merge (see
> `openspec/project.md` and `CONTRIBUTING.md`). Treat the committed
> `contracts/wasm/*.wasm` hash as part of that gate: if you rebuilt the WASM
> but did not commit the new bytes, the deploy path silently ships stale
> bytecode. Run `shasum -a 256 contracts/wasm/*.wasm` locally before opening
> the PR and paste the output into the PR description so a reviewer can
> cross-reference it.

---

## Manifest schema

Every file in `contracts/manifests/` is a single JSON object describing one
contract. The LLM catalog is built from these files (see
`openspec/project.md` → "Contracts" convention: *add a manifest JSON in
`contracts/manifests/`; the LLM catalog is built from these and injected into
the system prompt*).

The four manifests in this directory (`oz-ownable.json`,
`oz-fungible-token.json`, `oz-nft.json`, `soroswap-router.json`) all use the
same field set. Required and optional fields:

| Field | Required | Type | Notes |
| --- | --- | --- | --- |
| `id` | yes | string | Stable catalog id, e.g. `oz-fungible-token`. Used in the LLM prompt and in `argsFromConfig`. |
| `name` | yes | string | Human-readable contract name shown in the catalog UI. |
| `description` | yes | string | One-paragraph description of what the contract does and when to use it. |
| `useFor` | yes | string | Comma-separated list of natural-language use cases ("currencies, points, credits"). |
| `type` | yes | string | One of `deployable` (we ship the WASM) or `deployed` (we connect to an existing contract id). |
| `category` | yes | string | UI grouping — `access`, `token`, `nft`, `dex`, etc. |
| `wasmPath` | **only if `type`=`deployable`** | string | Repo-relative path to the committed `.wasm` file. The deploy path reads this. |
| `contractId` | **only if `type`=`deployed`** | string | Stellar contract id of the existing contract (e.g. Soroswap router on testnet). |
| `network` | only if `type`=`deployed`** | string | Which Stellar network the contract id lives on (e.g. `testnet`). |
| `init` | only if `type`=`deployable`** | object | Constructor invocation spec — see below. |
| `config` | yes | array | Ordered list of constructor/UI config fields — see below. |
| `methods` | yes | array | Public methods surfaced in the catalog UI — see below. |

### `init` block (deployable contracts only)

```json
"init": {
  "method": "__constructor",
  "argsFromConfig": ["name", "symbol", "owner", "initial_supply"]
}
```

- `method` is always `__constructor` for OpenZeppelin Soroban contracts.
- `argsFromConfig` is the **ordered** list of `config[].key` values that the
  deploy path passes to the constructor, in that exact order. The order must
  match the contract's Rust `__constructor(env, …)` signature.

`oz-ownable.json` uses `["owner"]`. `oz-fungible-token.json` uses
`["name", "symbol", "owner", "initial_supply"]`. `oz-nft.json` uses
`["uri", "name", "symbol", "owner"]`.

### `config` array

Each entry describes one field the LLM (or the user via the catalog UI) sets:

```json
{ "key": "initial_supply", "label": "Initial supply", "type": "number", "default": 1000000 }
```

| Sub-field | Required | Notes |
| --- | --- | --- |
| `key` | yes | Matches an entry in `init.argsFromConfig`. |
| `label` | yes | Human-readable label in the catalog UI. |
| `type` | yes | One of `string`, `number`, `address`. The deploy path converts these to ScVal (`string`→ScvString, `number`→ScvI128, `address`→ScvAddress, `u32`/`u64` for explicit sizes — see `toScVal()` in `deploy.mjs`). |
| `default` | no | Default value if the LLM omits this. `{{deployer}}` is special — it gets replaced with the ephemeral deployer keypair's public key. |

### `methods` array

Each entry surfaces one public method in the catalog:

```json
{ "name": "transfer", "args": ["from", "to", "amount"], "returns": "void", "mutates": true, "description": "Send tokens…" }
```

`mutates` (`true`/`false`) tells the LLM whether the method is a write or a
read — read-only methods can be called without a Freighter signature.

### `deployed` contracts (e.g. Soroswap router)

`type: "deployed"` contracts have no `wasmPath`, no `init`, and no `config` —
they only declare the existing `contractId`, `network`, and the `methods` the
LLM should call. The Soroswap router manifest (`soroswap-router.json`) is the
canonical example: `contractId` and `network` set, `config` empty, methods
documented with `description` so the LLM knows to quote first via
`router_get_amounts_out`, then swap via `swap_exact_tokens_for_tokens` with 5%
slippage.

---

## Dry-running `deploy.mjs`

`contracts/scripts/deploy.mjs` is a standalone validation of the pure-JS
deploy path — it mirrors `server/_lib/deploy.ts` exactly so you can prove the
SDK flow on testnet before wiring a new contract into the backend.

> **Note.** `deploy.mjs` submits **real testnet transactions**. There is no
> `--dry-run` flag. The script generates an ephemeral keypair, funds it via
> Friendbot, uploads the WASM, invokes `__constructor`, and prints the new
> contract address. Treat the printed `explorerUrl` as proof of success — the
> contract is real and on-chain (testnet only, no funds at risk).

### Usage

```bash
node contracts/scripts/deploy.mjs <wasmPath> '<configJson>' '<argsOrder>' '<typesJson>'
```

The four positional arguments are:

1. `wasmPath` — repo-relative path to the `.wasm` file (matches `wasmPath` in
   the manifest).
2. `configJson` — JSON object with one entry per `argsFromConfig` key.
3. `argsOrder` — comma-separated list of `argsFromConfig` keys, in the order
   they appear in the contract's Rust `__constructor` signature.
4. `typesJson` — JSON object mapping each arg name to its ScVal type. See
   `toScVal()` in `deploy.mjs` for the supported set (`string`, `address`,
   `i128`, `u32`, `u64`).

### Example: deploy the fungible token

```bash
node contracts/scripts/deploy.mjs contracts/wasm/fungible-token.wasm \
  '{"name":"Peña","symbol":"PENA","owner":"{{deployer}}","initial_supply":10000}' \
  'name,symbol,owner,initial_supply' \
  '{"name":"string","symbol":"string","owner":"address","initial_supply":"i128"}'
```

The script prints a JSON object with `contractId`, `deployer`, `txHash`, and
`explorerUrl` (a `stellar.expert` testnet link) on success.

### Notes

- The `{{deployer}}` placeholder is replaced with the ephemeral deployer
  keypair's public key — this matches what the Express backend does so the
  user's Freighter wallet becomes the contract owner in production.
- The script polls Soroban RPC until the Friendbot-funded account is visible,
  then retries on the transient `MissingValue` / "Wasm does not exist" errors
  that follow a fresh upload. Don't add a `setTimeout` retry loop outside
  this script — `submit()` and `getAccount()` already poll correctly.
- This script is **only for catalog validation**. Production deploys go
  through `server/_lib/deploy.ts` (Express endpoint). Do not call `deploy.mjs`
  from the browser.

---

## Adding a contract type

A new contract type needs three changes — a Rust crate that builds to WASM,
a manifest that exposes it, and a committed WASM file. Use only this README
and `openspec/project.md` (specifically the "Contracts" convention) as your
reference.

### Step 1 — Pick the source

OpenZeppelin `stellar-contracts` ships a curated set of deployable examples.
Pick one from `examples/` in that repo, or pin a different example crate by
adding it to `CATALOG` in `build.sh`:

```bash
# In build.sh, append: "<cargo-package-name>:<wasm-output-name>"
CATALOG=(
  "ownable-example:ownable"
  "fungible-pausable-example:fungible-token"
  "nft-sequential-minting-example:nft"
  "your-new-package:your-new-contract"   # ← new line
)
```

`<cargo-package-name>` is the package name in OpenZeppelin's `Cargo.toml`.
`<wasm-output-name>` is the filename that lands in `contracts/wasm/` — this
is what you will reference in the manifest's `wasmPath`.

### Step 2 — Rebuild

```bash
./contracts/build.sh
```

The script clones the pinned OpenZeppelin tag into `contracts/.oz-src/`,
compiles each `CATALOG` entry, and copies the emitted `.wasm` from
`$SRC/target/wasm32v1-none/release/${pkg//-/_}.wasm` to
`contracts/wasm/<name>.wasm`. Note the `wasm32v1-none` target — Soroban SDK
26 emits to that path, not `wasm32-unknown-unknown`.

### Step 3 — Commit the rebuilt WASM

```bash
git add contracts/wasm/your-new-contract.wasm
git add contracts/build.sh
```

The committed bytes are what the server deploys. The PR description should
include the SHA-256 of the new file (printed by `build.sh`) so a reviewer can
cross-reference the committed bytes against the build output. Run:

```bash
shasum -a 256 contracts/wasm/your-new-contract.wasm
```

and paste the output in the PR description.

### Step 4 — Write the manifest

Create `contracts/manifests/<id>.json`. Use the schema above. A minimal
`deployable` manifest:

```json
{
  "id": "oz-your-new-contract",
  "name": "Your New Contract",
  "description": "What it does and when to use it.",
  "useFor": "use, cases, here",
  "type": "deployable",
  "category": "your-category",
  "wasmPath": "contracts/wasm/your-new-contract.wasm",
  "init": {
    "method": "__constructor",
    "argsFromConfig": ["owner"]
  },
  "config": [
    { "key": "owner", "label": "Owner", "type": "address", "default": "{{deployer}}" }
  ],
  "methods": [
    { "name": "get_owner", "args": [], "returns": "address", "mutates": false, "description": "Read the current owner." }
  ]
}
```

The `argsFromConfig` order MUST match the Rust `__constructor` signature —
`deploy.mjs` (and `server/_lib/deploy.ts`) map each arg through `toScVal()`
in the order you list here. Mismatched order throws `XDR Write Error` on
testnet.

### Step 5 — Dry-run on testnet

Before opening the PR, prove the new contract actually deploys:

```bash
node contracts/scripts/deploy.mjs contracts/wasm/your-new-contract.wasm \
  '{"owner":"{{deployer}}"}' \
  'owner' \
  '{"owner":"address"}'
```

A `contractId` + `explorerUrl` in stdout means the new contract is live on
testnet. Paste that URL into the PR description as evidence.

### Step 6 — Quality gate

Before opening the PR:

```bash
pnpm typecheck && pnpm lint && pnpm build
```

All three must pass — see `openspec/project.md` and `CONTRIBUTING.md`. Commit
the rebuilt WASM in the same PR as the manifest + `build.sh` change.

### Step 7 — Open the PR

- Reference the issue, e.g. `Closes #76`.
- One issue per pull request.
- Paste the `shasum -a 256` output and the `deploy.mjs` `explorerUrl` in the
  description.
