---
name: develop-relayer
description: Use when the user wants to fork or extend the reference Rust axum relayer — custom storage backend (PostgreSQL, S3, …), custom auth middleware, new routes/handlers, sponsor-key strategy, allowlists, rate limiting, or running it inside Nautilus. Trigger phrases - "extend the relayer", "fork the relayer", "add storage to relayer", "custom auth middleware", "custom relayer handler", "PostgreSQL relayer", "relayer rate limiting".
---

# Develop the relayer

The reference relayer at `relayer/` is meant to be forked. It deliberately ships with in-memory storage and a fixed auth pipeline so you can substitute production-grade pieces.

## Mental model

You're forking a reference — not modifying a library. Don't worry about preserving an upstream API. Do preserve:

- **The wire protocol** the SDK speaks (`docs/sui-stack-messaging/Relayer.md`). Breaking this means forking the SDK too.
- **The auth invariants**: every write is a wallet signature over a canonical message, verified against on-chain group membership. If you change auth, the SDK must verify the same way.

Everything else — routes you add, storage you swap in, observability, deployment — is yours.

## Run + test loop

```bash
cd relayer

cargo run                                 # dev server :3000 (see spin-up-relayer)
cargo test                                # all tests, no network needed
cargo test --test auth_integration_test
cargo test --test membership_sync_test
cargo test --test walrus_sync_test
cargo test -- --ignored                   # tests that hit Walrus testnet
cargo fmt
cargo clippy --all-targets -- -D warnings
```

Toolchain: stable, with `clippy` and `rustfmt` (`rust-toolchain.toml`).

## Source layout

```
src/
├── main.rs                      binary entry; wires Router + services
├── lib.rs                       library entry
├── auth/
│   ├── middleware.rs            7-step verification pipeline
│   ├── membership.rs            on-chain membership lookups
│   └── schemes.rs               Ed25519 / Secp256k1 / Secp256r1 signature dispatch
├── handlers/messages/handlers.rs  CRUD endpoints
├── storage/
│   ├── adapter.rs               StorageAdapter trait
│   └── memory.rs                in-memory impl (default)
└── services/
    ├── membership_sync.rs       gRPC checkpoint listener
    ├── walrus_sync.rs           batched Walrus archival
    └── event_parser.rs
```

## Common extensions

### 0. Checkpoint backfill / resume (usually first)

The reference relayer does not backfill. `MembershipSyncService` (`src/services/membership_sync.rs`) starts with `last_cursor: None` and subscribes to the live gRPC checkpoint stream — events emitted while the relayer was down are dropped, and on restart membership state begins from "now." A fresh relayer therefore has an empty `MembershipStore` and will reject every authenticated write until the on-chain events it cares about fire again. Fix this before any deployment that survives restarts.

The canonical pattern is to walk historical checkpoints via the same gRPC service the live tail uses, not via paginated RPC event queries. Shape:

1. **Persist a cursor pair** `(checkpoint_sequence, last_processed_tx_digest)` after each processed checkpoint. The tx digest is needed because checkpoints contain many transactions, and on resume you must skip the ones already processed in the boundary checkpoint.
2. **On startup**, read the persisted cursor and the current chain tip. Walk forward one checkpoint at a time using gRPC `LedgerService.GetCheckpoint(sequence_number)`. For the first (boundary) checkpoint, skip transactions until you pass `last_processed_tx_digest`; for every subsequent checkpoint, process every transaction normally. Persist the cursor after each one so a crash mid-backfill resumes correctly.
3. **Then** subscribe to `SubscriptionService.SubscribeCheckpoints` for the live tail and continue updating the cursor.
4. **On reconnect** to the live tail, repeat step 2 from the persisted cursor up to the new tip before re-joining the stream. This handles transient disconnects without dropping events.

The same pattern applies to `WalrusSyncService` if you care about archiving messages that arrived while the relayer was down — though there `last_cursor` is per-message in the storage adapter, so the gap is naturally smaller.

### 1. Custom storage backend (PostgreSQL / Redis / S3)

Implement the trait in `src/storage/adapter.rs`. The in-memory impl in `src/storage/memory.rs` is your reference.

```rust
// new file: src/storage/postgres.rs
use crate::storage::adapter::StorageAdapter;

pub struct PostgresStorage { /* pool */ }

#[async_trait::async_trait]
impl StorageAdapter for PostgresStorage {
    // implement create/get/update/delete + sync-status methods
}
```

Then dispatch on `STORAGE_TYPE` env var in `main.rs`:

```rust
let storage: Arc<dyn StorageAdapter> = match env::var("STORAGE_TYPE").as_deref() {
    Ok("memory") | _ => Arc::new(InMemoryStorage::new()),
    Ok("postgres") => Arc::new(PostgresStorage::new(/* … */).await?),
};
```

The same pattern applies to `MEMBERSHIP_STORE_TYPE` (`src/auth/membership.rs`).

### 2. Custom auth / allowlist / rate limit

`src/auth/middleware.rs` is the 7-step pipeline. To add an allowlist:

- Insert a check after membership verification but before handler dispatch.
- Use an axum `from_fn_with_state` middleware so you can read shared config.

For rate limiting, layer `tower::limit::RateLimitLayer` on the router in `main.rs`.

### 3. New endpoint

Add a handler under `src/handlers/`, register on the `Router` in `main.rs`. If the endpoint mutates state, run it through the same auth middleware so signatures are verified consistently.

### 4. Sponsor-key / gas strategy

The reference relayer **does not** sponsor transactions today — clients sign their own. Adding sponsorship to the relayer is a non-trivial design choice (sponsor key custody, abuse limits, dry-run policy, refund flows) and there is no in-repo blueprint for it.

If you need sponsored-tx UX for onboarding or smoother flows, **Mysten Enoki** is an option. See https://enoki.mystenlabs.com/

### 5. New signature scheme

`src/auth/schemes.rs` enumerates supported schemes. Add a variant + dispatch arm. Touch points: scheme parsing, verification, error mapping.

**zkLogin support is a known gap** — the relayer's current schemes are Ed25519, Secp256k1, Secp256r1 only. See [GitHub issue #63](https://github.com/MystenLabs/sui-stack-messaging/issues/63) for the discussion.

### 6. Observability

What's actually shipped today is minimal: `tracing` + `tracing-subscriber` are in `Cargo.toml`, `main.rs:39` calls `tracing_subscriber::fmt::init()`, and there are a handful of `tracing::info!` / `tracing::debug!` log lines (in `main.rs`, `config.rs`, `auth/membership.rs`). Log level is controlled by `RUST_LOG` (default `messaging_relayer=info`).

What's **not** there: no `#[instrument]` attributes on handlers or services, no explicit spans, no metrics, no OpenTelemetry wiring. So a real observability story for a fork involves:

- Annotate handlers and service methods with `#[tracing::instrument(skip(...))]` to get per-request and per-service spans automatically.
- Swap `tracing_subscriber::fmt::init()` for a registry that combines `EnvFilter` + `fmt` + a `tracing-opentelemetry` layer (with an OTLP exporter via `opentelemetry-otlp`) if you want distributed traces.
- For metrics, add `metrics` + `metrics-exporter-prometheus` (or similar) and instrument hot paths in handlers/services.

This is greenfield work — pick the stack your deployment uses and follow the standard Rust patterns. There's no in-repo precedent to mirror.

## Load-bearing surfaces — change only if you accept the cost

These three things are wire contracts shared with the SDK and the indexer. The default advice is "don't change them," but if you do, here's what falls out:

- **HTTP request/response shapes** in `handlers/messages/handlers.rs` are the contract `HTTPRelayerTransport` in the SDK speaks. If you change them, you must either:
  - Fork the SDK's `HTTPRelayerTransport` (`ts-sdks/packages/sui-stack-messaging/src/relayer/http-transport.ts`) to match — you're now on a SDK-side fork too, and need to track upstream changes to canonical wire fields (versioning, sender verification metadata, etc.) by hand.
  - Or implement a custom `RelayerTransport` (the public TS interface in `ts-sdks/packages/sui-stack-messaging/src/relayer/transport.ts`) and pass it via `relayer: { transport: myTransport }` instead of `relayer: { relayerUrl }`. This keeps the SDK upstream-clean but you own the bridging code on the client.

- **Canonical-message format used for signature verification** must match the SDK's `buildCanonicalMessage` / `verifyMessageSender` (`ts-sdks/packages/sui-stack-messaging/src/verification.ts`) byte-for-byte. If you change this, you change a security-critical surface and you must update the SDK in lockstep — otherwise signatures the SDK produces won't verify on the relayer (silent auth failures) or vice versa (forgeable writes). If you fork the SDK, this is also what *every other group member* uses to independently verify a message's sender; diverging means messages from your relayer's clients won't be verifiable by canonical-SDK clients in the same group.

- **Walrus archive format / patch naming** in `relayer/src/services/walrus_sync.rs` is consumed by two readers, not one:
  - `walrus-discovery-indexer/` (which inspects certified blobs and exposes them via REST).
  - The SDK's `RecoveryTransport` (`ts-sdks/packages/sui-stack-messaging/src/recovery/`), which uses the indexer's output to reconstruct message history.
  
  Changing the format means coordinated updates across all three sides. Existing archives written under the old format also stop being readable unless you handle migration explicitly. Treat this as a versioned contract — bump a format version and support both during transition rather than flipping the schema.

If you find yourself wanting to change one of these to fix an underlying problem, the usual better answer is to add a *new* surface alongside (a new endpoint, a new permission type, a new archive variant) rather than mutating an existing one — keeps you on the upstream upgrade path.

## Deployment notes

- `Dockerfile` builds a release binary; `docker-compose.yml` is dev-only (single container, no network).
- For Nautilus-attested deployments, see the architecture note in the root `README.md` ("Architecture Evolution") and `docs/sui-stack-messaging/Security.md`.
- `RUST_LOG` controls log level; default is `messaging_relayer=info`.

## Cross-links

- Run-local-only flow: [`spin-up-relayer`](../spin-up-relayer/SKILL.md).
- Full-stack with chat-app: [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md).
- Relayer protocol: `docs/sui-stack-messaging/Relayer.md`.
- Trust model (read before re-implementing auth): `docs/sui-stack-messaging/Security.md`.
- Relayer deep-dive: `relayer/README.md`.
