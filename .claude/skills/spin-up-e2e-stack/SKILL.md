---
name: spin-up-e2e-stack
description: Use when the user wants to run the full local stack end-to-end — relayer + walrus-discovery-indexer + chat-app — together for development or demo. Trigger phrases - "run everything locally", "full local stack", "end-to-end setup", "run the chat app with relayer and indexer", "demo setup", "spin up the whole thing".
---

# Spin up the full E2E stack

Three services, three terminals. Run in this order so each service is ready before the next one connects.

```
                       ┌────────────────────┐
┌─────────────┐        │  relayer (:3000)   │ ──▶ Sui RPC (gRPC) + Walrus testnet
│  chat-app   │ ─────▶ └────────────────────┘
│  (:5173)    │ ─────▶ Sui GraphQL (group discovery)
└─────────────┘
                       ┌────────────────────┐
                       │ walrus-indexer     │ ──▶ Sui RPC (gRPC) + Walrus testnet
                       │ (:3001, optional)  │     (consumed by SDK RecoveryTransport,
                       └────────────────────┘      not by chat-app)
```

## Prereqs

- Rust stable, Node + pnpm `>=10.17.0`, Docker (optional, only if running relayer in container).
- Working internet to reach Sui testnet RPC and Walrus testnet endpoints.

> **Use testnet, not mainnet, for this whole flow.** This skill assumes a dev wallet funded from the testnet faucet. Every "Create a group" / "Send a message" in the smoke test below mints real on-chain state on whichever network your relayer + chat-app are pointed at. Mainnet group/message creation costs real SUI, persists permanently, and may be visible to real users if your chat-app's group-discovery surface exposes it. Keep `GROUPS_PACKAGE_ID` (relayer) + `VITE_*` package configs (chat-app) on testnet for development.

## Recommended path: Docker for the two backend services

**Prefer Docker for the relayer and the walrus-indexer.** Both ship Dockerfiles; containers avoid host-toolchain friction. Steps 1–3 below are the no-Docker (host) alternative. The chat-app is a Vite dev server with no Dockerfile, so it always runs on the host (Step 3).

```bash
# relayer  (terminal 1)
cd relayer && cp .env.example .env          # set GROUPS_PACKAGE_ID (testnet id below)
docker compose up -d                         # :3000

# walrus-indexer  (terminal 2)
cd ../walrus-discovery-indexer && cp .env.example .env   # NETWORK=testnet (default)
docker build -t walrus-indexer .             # on pnpm v11 see the note below if this fails
docker run --rm -p 3001:3001 --env-file .env walrus-indexer   # :3001

# chat-app  (terminal 3, host) — see Step 3
```

> **If `docker build` fails** with `ERR_PNPM_IGNORED_BUILDS` (esbuild) or `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH` (overrides): you're on pnpm v11, which this repo isn't configured for yet (it targets `pnpm >=10.17.0`). Simplest fix — change `corepack prepare pnpm@latest` to `corepack prepare pnpm@10` in `walrus-discovery-indexer/Dockerfile`, then rebuild. Full explanation + the pnpm migration guide in [Troubleshooting](#troubleshooting).

## Step 1 — Relayer (terminal 1)

See [`spin-up-relayer`](../spin-up-relayer/SKILL.md) for full details.

```bash
cd relayer
cp .env.example .env
# set GROUPS_PACKAGE_ID for testnet:
#   0xba8a26d42bc8b5e5caf4dac2a0f7544128d5dd9b4614af88eec1311ade11de79
cargo run
```

Wait for: `listening on 0.0.0.0:3000` and the membership-sync log line.

## Step 2 — Walrus discovery indexer (terminal 2)

> Docker (above) is the recommended way to run this service. This repo targets **pnpm >=10.17.0**; on **pnpm v11** the host `pnpm install` below trips the esbuild build gate (`ERR_PNPM_IGNORED_BUILDS`). Run `pnpm install --ignore-scripts` instead — no change to your global toolchain. (Don't `corepack … --activate` a different pnpm globally just for this; it changes the default for all your projects.) See [Troubleshooting](#troubleshooting).

```bash
cd walrus-discovery-indexer
pnpm install
cp .env.example .env
# set NETWORK=testnet (already default)
pnpm dev                # tsx watch on src/index.ts
```

Listens on `:3001` by default (`PORT` env var). Subscribes to Sui checkpoint stream and inspects certified Walrus blobs. No database — in-memory store.

Health: `curl http://localhost:3001/health` (returns last processed checkpoint + archived-blob discovery stats — see `walrus-discovery-indexer/src/api.ts:62`).

## Step 3 — Chat-app (terminal 3)

The chat-app can resolve the SDK two ways. Almost everyone wants Mode A.

**Mode A — published SDK (default; use this unless you are changing the SDK itself).** Depend on the npm package: no `ts-sdks/` build, no `link:`, and the canonical workspace is never touched. In `chat-app/package.json` set the dependency to the published version (currently `0.0.2`):

```jsonc
// chat-app/package.json
"@mysten/sui-stack-messaging": "0.0.2"   // was: link:../ts-sdks/packages/sui-stack-messaging
```

```bash
cd chat-app
cp .env.example .env       # defaults point to localhost relayer + testnet Walrus
pnpm install               # no build:deps needed
# pnpm v11: if this trips the esbuild gate (ERR_PNPM_IGNORED_BUILDS, via vite), add the flag:
#   pnpm install --ignore-scripts          (see Troubleshooting)
pnpm dev                   # vite on :5173
```

> Mode A only swaps the dependency; the `build` script still chains `build:deps` (`"build": "pnpm build:deps && tsc -b && vite build"`), which rebuilds the canonical `ts-sdks/` SDK you no longer depend on and **fails on pnpm v11** (the `ts-sdks/` `--frozen-lockfile` install hits `ERR_PNPM_*`). `pnpm dev` is unaffected — it doesn't run `build:deps`. For a production build in Mode A, drop `build:deps` from the `build` script in your fork (`"build": "tsc -b && vite build"`).

**Mode B — linked workspace SDK (repo maintainers, or open-source contributors trying changes to the canonical `ts-sdks`).** Not for app builders — only use this if you are developing `@mysten/sui-stack-messaging` itself inside this repo. Keep the `link:../ts-sdks/...` dependency and build it first:

```bash
cd chat-app
cp .env.example .env
pnpm install
pnpm build:deps            # builds @mysten/sui-stack-messaging from the ts-sdks/ workspace
pnpm dev                   # vite on :5173
```

> `build:deps` runs `pnpm install --frozen-lockfile` inside `ts-sdks/`, which on pnpm v11 fails with `ERR_PNPM_*` (the canonical workspace still uses pnpm-v10 config, and `--frozen` surfaces the `overrides` mismatch). Don't edit `ts-sdks/` (canonical) and don't globally switch your pnpm. Build the SDK with a project-scoped pnpm 10 (e.g. `npx pnpm@10 …`), or just use Mode A. See [Troubleshooting](#troubleshooting).

Open http://localhost:5173. Connect a wallet (testnet) with some testnet SUI for gas.

## Env wiring at a glance

| Service        | Port | Reads from                   | Where configured                                              |
| -------------- | ---- | ---------------------------- | ------------------------------------------------------------- |
| relayer        | 3000 | Sui RPC, Walrus              | `relayer/.env` (`SUI_RPC_URL`, `GROUPS_PACKAGE_ID`)           |
| walrus-indexer | 3001 | Sui RPC (gRPC), Walrus       | `walrus-discovery-indexer/.env` (`NETWORK`)                   |
| chat-app       | 5173 | relayer, Sui GraphQL, Walrus | `chat-app/.env` (`VITE_RELAYER_URL=http://localhost:3000`, …) |

The chat-app does **not** interact with the `walrus-discovery-indexer` at all — there is no `VITE_INDEXER_URL` in `chat-app/.env.example` and no indexer references in `chat-app/src/`. The indexer is started in this skill purely so the full stack runs in parallel; you can skip Step 2 entirely if you only want to use the chat-app.

Two distinct concerns that both contain the word "discovery" but are unrelated:

- **Group Discovery** (finding `PermissionedGroup<Messaging>` objects the user is a member of) — handled by the chat-app via Sui GraphQL queries against `VITE_SUI_GRAPHQL_URL`. See `docs/sui-stack-messaging/GroupDiscovery.md`.
- **Archived-message recovery** (locating Walrus blobs/quilts the relayer wrote during archival) — what `walrus-discovery-indexer` is for. The SDK's `RecoveryTransport` is the consumer (not the chat-app). See `docs/sui-stack-messaging/ArchiveRecovery.md`.

## Containers — detail and caveats

This expands the [recommended Docker path](#recommended-path-docker-for-the-two-backend-services) at the top. Both backend services ship Dockerfiles, so you can run the stack without a Rust/Node toolchain on the host — also how you'd mimic a deployment topology or run from CI.

```bash
# Relayer — already has docker-compose
cd relayer
cp .env.example .env                        # set GROUPS_PACKAGE_ID
docker compose up -d                        # :3000, healthcheck on /health_check (see docker-compose.yml)

# Walrus indexer — single Dockerfile (no compose)
cd ../walrus-discovery-indexer
cp .env.example .env                        # set NETWORK=testnet
docker build -t walrus-indexer .            # pnpm v11 fails here; build on pnpm 10 (corepack prepare pnpm@latest -> pnpm@10 in the Dockerfile). See Troubleshooting
docker run --rm -p 3001:3001 --env-file .env walrus-indexer
# image exposes :3001 with HEALTHCHECK on /health (see walrus-discovery-indexer/Dockerfile)
```

The chat-app stays as `pnpm dev` for hot reload during development. If you want it containerized too, build once with `pnpm build` and serve `dist/` behind any static webserver — there's no Dockerfile shipped for it.

Caveats:

- **Inter-container networking** is not pre-wired. If you put both services on the same Docker network, the chat-app's `VITE_RELAYER_URL` (set at build time) and the relayer-to-Sui gRPC connection both have to be configured for the topology you choose. Running the chat-app on the host while the backend services are containerized is the simplest combination.

## Build the SDK once when working across packages

```bash
cd chat-app
pnpm build:deps          # rebuilds the linked SDK package
# vite will hot-reload after this
```

`chat-app/package.json` links the SDK as `link:../ts-sdks/packages/sui-stack-messaging`

## Quick smoke test

1. Connect wallet in chat-app (top right).
2. Create a group → wait for tx confirmation.
3. Send a message → relayer logs `POST /messages 200`.
4. Refresh, message reappears (read path also OK).

## Troubleshooting

- **chat-app shows "relayer unreachable"** — relayer not running, or CORS misconfig (relayer permits localhost by default; check `tower-http` cors layer in `relayer/src/main.rs`).
- **After relayer restart, the chat-app still lists old groups but interactions fail** — expected with the reference relayer. Group existence is on-chain, so the chat-app's GraphQL-backed discovery still surfaces every group the wallet is a member of. The relayer, however, does **not** backfill: its membership store starts empty on each launch and is rebuilt only from Sui gRPC checkpoints emitted _after_ startup. Until membership events for a given group fire again on-chain, the relayer will reject reads/writes against it. Workaround: implement checkpoint backfill in your fork — see [`develop-relayer`](../develop-relayer/SKILL.md) and the same caveat in [`develop-on-sui-stack-messaging`](../develop-on-sui-stack-messaging/SKILL.md).
- **pnpm v11 fails a fresh install** — `ERR_PNPM_IGNORED_BUILDS` (esbuild) and/or `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH` (`overrides`), on the indexer and chat-app (relayer is Rust, unaffected). pnpm v11 no longer reads this repo's `package.json` `pnpm` config. **Quick fix:** Docker → pin `pnpm@10` in the Dockerfile; host → `pnpm install --ignore-scripts` (don't globally switch pnpm). Root cause, both errors, all remedies, and the pnpm migration guide: [`docs/pnpm-v11-troubleshooting.md`](../../../docs/pnpm-v11-troubleshooting.md).
- **Wallet has no SUI** — fund via `https://faucet.testnet.sui.io`

## Want to deploy each piece separately?

- Relayer alone: [`spin-up-relayer`](../spin-up-relayer/SKILL.md).
- Customize relayer behavior: [`develop-relayer`](../develop-relayer/SKILL.md).
- Customize walrus indexer: [`develop-walrus-indexer`](../develop-walrus-indexer/SKILL.md).
- Add Move-side functionality: [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md).
