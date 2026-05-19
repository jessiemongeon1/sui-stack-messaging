# AGENTS.md

This file covers the Move smart-contracts subtree (`move/`). Root-level guidance still applies — `../AGENTS.md` is loaded transitively via `../CLAUDE.md` — this file only adds the commands and invariants that are Move-specific.

## Repo layout (this subtree)

```
move/
  packages/sui_stack_messaging/    canonical — published mainnet+testnet
  packages/example_app/             reference extensions (custom_seal_policy, paid_join_rule)
  design_docs/                      design documentation
```

## Commands

```bash
sui move build --path packages/sui_stack_messaging
sui move test  --path packages/sui_stack_messaging
sui move build --path packages/example_app
sui move test  --path packages/example_app
```

## Toolchain

- Move edition: `2024`.

## Hard invariants

- **Do not hand-edit `packages/sui_stack_messaging/Published.toml`.** It is the committed source of truth for deployed package IDs on mainnet/testnet. Re-publishing is a maintainer-only action via the `publish/` helper.
- **Do not fork the canonical package.** `packages/sui_stack_messaging/` is already published as `0xcbd2f4c25c...` (mainnet) and `0x047696be0e...` (testnet). To add behavior, write your own package that depends on `sui_stack_messaging` — see `packages/example_app/` for two worked examples (custom seal policy + paid join rule).
- **Do not redefine canonical permission types** (`MessagingSender`, `MessagingReader`, `MessagingEditor`, `MessagingDeleter`, `MetadataAdmin`, `SuiNsAdmin`) in extension packages. Add new types alongside; the canonical ones are already wired into the published contracts and into `sui-groups`.
- **Do not change the Seal identity-bytes layout** (`[group_id (32)][key_version (8 LE u64)]`). It is shared with the SDK and the relayer; changing it on the Move side silently breaks decryption everywhere else.

## Where extensions live

Use the `extend-smart-contracts` skill (`../.claude/skills/extend-smart-contracts/SKILL.md`) for the recipe. Worked examples:

- `packages/example_app/sources/custom_seal_policy.move` — defines a fresh permission type and a `seal_approve_*` entry that gates decryption on app-specific state.
- `packages/example_app/sources/paid_join_rule.move` — adds a paid-join policy alongside the canonical permission types without redefining them.

## Documentation pointers

- `design_docs/REQUIREMENTS.md` — high-level requirements.
- `design_docs/sui_stack_messaging/*.md` — module-by-module design.
- `../docs/sui-stack-messaging/Encryption.md` and `../docs/sui-stack-messaging/Security.md` — load-bearing crypto invariants that touch Move struct layouts; read before changing anything in `sources/seal_policies.move`, `auth.move`, or the message/attachment structs.

## Pre-commit checks

```bash
sui move build --path packages/sui_stack_messaging
sui move test  --path packages/sui_stack_messaging
sui move build --path packages/example_app
sui move test  --path packages/example_app
# If the public Move API changed (new entry/public functions, struct fields,
# event shapes), also regenerate the SDK bindings so the TS side stays in sync:
cd ../ts-sdks/packages/sui-stack-messaging && pnpm codegen
```
