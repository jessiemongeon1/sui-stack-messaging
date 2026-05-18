# AGENTS.md — publish/

**This subtree is for repo maintainers only.** It contains a Node.js helper used to publish the canonical `sui_stack_messaging` Move package to mainnet and testnet. The helper either signs and submits the publish transaction directly (development path), or emits unsigned transaction bytes for a multisig / KMS signer (production path). The resulting package IDs and `UpgradeCap` belong to the maintainer team and must be reflected back into `move/packages/sui_stack_messaging/Published.toml` by hand.

**If you are a downstream developer** (Builder or contributor): you almost certainly do not need this directory. To publish *your own* Move package that *depends on* `sui_stack_messaging`, use `sui client publish` directly from your package's directory. See `.claude/skills/extend-smart-contracts/SKILL.md`.

## Repo layout

```
publish/
  src/
    env.ts                      env parsing (zod)
    scripts/publish.ts          publish + sign + submit (dev path)
    scripts/publishBytes.ts     emit unsigned tx bytes (prod / multisig path)
    utils/                      shared helpers
  package.json
  .env.example                  required maintainer env
  README.md                     operational walkthrough — read before running
```

## When and how this gets used

- **Initial publish**: maintainer runs the helper after the Move package is reviewed and ready for mainnet/testnet.
- **Upgrade**: a fresh publish via this helper produces a new package ID and `UpgradeCap`; the existing `UpgradeCap` flow (handover to the maintainer multisig, on-chain upgrade) is performed separately via `sui client` once the cap is held by the right account.
- The helper operates on whatever path `MOVE_PACKAGE_PATH` points at — in this repo that should be `../move/packages/sui_stack_messaging`. After a successful publish, the maintainer updates `../move/packages/sui_stack_messaging/Published.toml` with the new IDs.

## Hard invariants

- **Do not run this from a contributor flow.** It mints on-chain state and (in the dev path) requires the production sponsor key. CI doesn't and shouldn't invoke it.
- **Do not refactor `../move/packages/sui_stack_messaging/Published.toml`** to a different format — this helper's output, the SDK constants, and Move builds all read the current format.
- **Do not commit `.env`** — required maintainer secrets (`ADMIN_SECRET_KEY`, fullnode URL) live there; only `.env.example` is checked in.
- **Do not point `MOVE_PACKAGE_PATH` at the wrong package.** Double-check before running mainnet.

## Documentation pointers

- [`README.md`](./README.md) — operational checklist for both publish paths.
- [`../move/AGENTS.md`](../move/AGENTS.md) — Move-side invariants this helper respects.
- [`../docs/sui-stack-messaging/`](../docs/sui-stack-messaging/) — SDK and Move design docs (broader context).

## Commands (maintainers only)

From `package.json`:

- `pnpm run deploy` — build + publish + sign + submit. Requires `ADMIN_SECRET_KEY`, `SUI_FULLNODE_URL`, `MOVE_PACKAGE_PATH`. Writes the publish response to `data/publish.json`.
- `pnpm run deploy-bytes` — build the publish tx and emit unsigned base64 bytes to `data/publish-bytes.txt` for offline multisig / KMS signing. Requires `ADMIN_ADDRESS`, `SUI_FULLNODE_URL`, `MOVE_PACKAGE_PATH` (no secret key needed).
