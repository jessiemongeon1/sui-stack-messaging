# AGENTS.md — walrus-discovery-indexer

This is the **reference** TypeScript indexer for Walrus blobs published by the messaging relayer. It is designed to be forked and extended — custom event filters, a persistent storage backend in place of the in-memory default, and output sinks like webhooks or queues. Root-level guidance in [`../AGENTS.md`](../AGENTS.md) still applies; this file adds indexer-specific commands and invariants.

## Repo layout (this subtree)

```
walrus-discovery-indexer/
  src/                          event-pull, filter pipeline, in-memory storage, REST API
    index.ts                    entrypoint
    config.ts                   env loading
    constants.ts                Walrus package IDs per network
    checkpoint-listener.ts      event source (tier 1)
    event-parser.ts             BCS decode of Walrus events
    blob-inspector.ts           filter pipeline + blob fetch
    discovery-store.ts          in-memory storage (swap for prod)
    api.ts                      Express REST API
    types.ts                    shared types
  docs/                         indexer-specific docs
  Dockerfile                    container image
  package.json                  pnpm scripts
  .env.example                  required env
  README.md                     ~130 line reference
```

## Commands

```bash
pnpm install
pnpm dev                       # tsx watch src/index.ts
pnpm build                     # tsc -> dist/
pnpm start                     # node dist/index.js
pnpm test                      # vitest run
```

## Required env (.env)

Translate from `.env.example`:

- `NETWORK` — required, `testnet` or `mainnet`. Selects the Walrus package ID and the Sui fullnode gRPC URL.
- `WALRUS_PUBLISHER_SUI_ADDRESS` — optional tier-1 sender filter; without it, the indexer inspects every certified blob (noisy). **Gotcha:** `.env.example` ships this uncommented as the literal placeholder `0x...`, so a plain `cp .env.example .env` makes the indexer filter on the string `0x...` (it logs `Sender filter active: 0x...` and matches nothing). Comment it out / leave it empty, or set a real publisher address.
- `PORT` — optional REST API port, default `3001`.

The fullnode gRPC URL is derived from `NETWORK` in `src/config.ts` and is not env-configurable today.

## Toolchain

- pnpm (root constraint `>=10.17.0`) — **use pnpm 10.x.** On pnpm v11 a fresh `pnpm install` / `docker build` fails (`ERR_PNPM_IGNORED_BUILDS`, `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`); see [`../docs/pnpm-v11-troubleshooting.md`](../docs/pnpm-v11-troubleshooting.md).
- TypeScript 5.7, ESM-only (`"type": "module"`)
- vitest for tests
- Dev loop: tsx watch (`pnpm dev`)

## Hard invariants

- **Do not break the wire protocol** between the relayer and the indexer. The indexer parses what the relayer writes to Walrus; types must stay in sync. Changes here must update the relayer + SDK + `../docs/sui-stack-messaging/Relayer.md` together. (Also enforced as a path-scoped rule under `.claude/rules/`.)
- **Preserve the 3-tier filter pipeline shape** when forking: event source -> filter chain -> output sink. The pipeline boundary is the extensibility contract; replace tiers, don't dissolve the boundary. See `README.md` for the canonical shape.
- **Default storage is in-memory** (`discovery-store.ts`) — fine for dev, not for production. Forks for production must swap this for a persistent backend (PostgreSQL, MongoDB) at the storage interface boundary, not by introducing storage calls throughout the pipeline.

## When to fork vs configure

- Fork this code if you need: custom event filters not expressible via config, persistent storage, output sinks (webhooks, queues), or BCS event parsing for a different upstream package.
- Use as-is (locally or in production) if you only need the default Walrus discovery surface for `sui_stack_messaging`.

## Documentation pointers

- `README.md` (this directory) — feature overview + REST API.
- `docs/README.md` (this directory) — additional reference material.
- `../docs/sui-stack-messaging/GroupDiscovery.md` — feature deep-dive on group discovery via Walrus blobs.
- `../docs/sui-stack-messaging/ArchiveRecovery.md` — recovery flow this indexer supports.

## Skills

- `.claude/skills/spin-up-e2e-stack/SKILL.md` — full local stack with the indexer.
- `.claude/skills/develop-walrus-indexer/SKILL.md` — fork-and-extend recipe.

## Pre-commit checks

```bash
pnpm build                     # tsc typecheck + emit
pnpm test                      # vitest unit
```
