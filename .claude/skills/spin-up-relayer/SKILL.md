---
name: spin-up-relayer
description: Use when the user wants to run the reference messaging relayer locally — cargo run, docker compose, configuring the .env file, picking testnet vs mainnet GROUPS_PACKAGE_ID. Trigger phrases - "run the relayer", "start the relayer locally", "spin up the relayer", "relayer cargo run", "relayer docker", "relayer .env", "messaging relayer setup".
---

# Spin up the relayer locally

The relayer is at `relayer/` — a Rust axum service. Reference implementation; safe to run as-is for dev.

## Prereqs

- Rust stable (toolchain pinned in `relayer/rust-toolchain.toml`).
- A deployed `sui_stack_messaging` package ID for the network you point at. The canonical deployments are listed in `relayer/.env.example`:
  - Testnet: `0xba8a26d42bc8b5e5caf4dac2a0f7544128d5dd9b4614af88eec1311ade11de79`
  - Mainnet: `0x541840ae7df705d1c6329c22415ed61f9140a18b79b13c1c9dc7415b115c1ba8`
  - These are the **sui-groups** package IDs the relayer reads — confirm against `relayer/.env.example` if it changes.

## Configure

```bash
cd relayer
cp .env.example .env
# edit .env: set GROUPS_PACKAGE_ID (required), pick SUI_RPC_URL
```

Required env vars (from `.env.example`):
- `SUI_RPC_URL` — Sui fullnode endpoint (defaults to `https://fullnode.testnet.sui.io:443`).
- `GROUPS_PACKAGE_ID` — sui-groups package on the target network.

Optional (defaults shown in `.env.example`):
- `PORT` (3000), `REQUEST_TTL_SECONDS` (900).
- `STORAGE_TYPE` (`memory`), `MEMBERSHIP_STORE_TYPE` (`memory`).
- `WALRUS_PUBLISHER_URL`, `WALRUS_AGGREGATOR_URL`, `WALRUS_STORAGE_EPOCHS`, `WALRUS_SYNC_INTERVAL_SECS`, `WALRUS_SYNC_BATCH_SIZE`, `WALRUS_SYNC_MESSAGE_THRESHOLD`.
- `RUST_LOG=messaging_relayer=info` (use `=debug` for verbose).

## Run with cargo

```bash
cargo run                       # binds :3000 by default
PORT=8080 cargo run             # override port
RUST_LOG=debug cargo run        # verbose logs
```

Health check:

```bash
curl http://localhost:3000/health_check
```

## Run with Docker

```bash
docker compose up               # foreground; reads .env automatically
docker compose up -d            # detached
docker compose logs -f          # follow logs
docker compose down             # stop
```

Compose file: `relayer/docker-compose.yml`. Image build: `relayer/Dockerfile`.

## Sanity check

The relayer is up when:
1. `curl :3000/health_check` returns 200.
2. Logs show membership-sync subscribed to the Sui gRPC checkpoint stream.
3. The chat-app or another SDK client successfully posts a message and gets it back via GET.

## Storage note

`STORAGE_TYPE=memory` means messages are lost on restart. Fine for dev. For persistent storage, see [`develop-relayer`](../develop-relayer/SKILL.md) — implement the `StorageAdapter` trait.

## What this relayer does

In one sentence: authenticates per-message wallet signatures against on-chain group membership, stores E2E-encrypted message payloads off-chain, and periodically archives them to Walrus. It never sees plaintext.

Full protocol: `docs/sui-stack-messaging/Relayer.md` and `relayer/README.md`. Postman collection: `relayer/docs/messaging-relayer.postman_collection.json`.

## Common issues

- **`GROUPS_PACKAGE_ID` empty** — the relayer will fail to start. Set it.
- **Membership sync not catching up** — verify `SUI_RPC_URL` supports gRPC on port 443, and that `GROUPS_PACKAGE_ID` matches the network the RPC points to.
- **Walrus calls failing** — testnet Walrus endpoints are public but rate-limited; for sustained dev work, run your own publisher/aggregator.

## Next steps

- Want to run with the chat-app and indexer? → [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md).
- Want to fork and extend (custom storage, auth, handlers)? → [`develop-relayer`](../develop-relayer/SKILL.md).
