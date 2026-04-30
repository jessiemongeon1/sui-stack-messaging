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

```bash
cd chat-app
cp .env.example .env
# defaults already point to localhost relayer + testnet Walrus
pnpm install
pnpm build:deps          # builds @mysten/sui-stack-messaging from ts-sdks/ workspace
pnpm dev                 # vite on :5173
```

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

## Alternative — run as containers

Both backend services ship Dockerfiles, so you can stand up the same stack without a Rust/Node toolchain on the host. Use this when you want to mimic a deployment topology or run from CI.

```bash
# Relayer — already has docker-compose
cd relayer
cp .env.example .env                        # set GROUPS_PACKAGE_ID
docker compose up -d                        # :3000, healthcheck on /health_check (see docker-compose.yml)

# Walrus indexer — single Dockerfile (no compose)
cd ../walrus-discovery-indexer
cp .env.example .env                        # set NETWORK=testnet
docker build -t walrus-indexer .
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
- **`pnpm build:deps` errors** — run from `chat-app/` only; that's the script that knows the workspace path.
- **Wallet has no SUI** — fund via `https://faucet.testnet.sui.io`

## Want to deploy each piece separately?

- Relayer alone: [`spin-up-relayer`](../spin-up-relayer/SKILL.md).
- Customize relayer behavior: [`develop-relayer`](../develop-relayer/SKILL.md).
- Customize walrus indexer: [`develop-walrus-indexer`](../develop-walrus-indexer/SKILL.md).
- Add Move-side functionality: [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md).
