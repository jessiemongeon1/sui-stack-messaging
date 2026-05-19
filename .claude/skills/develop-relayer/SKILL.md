---
name: develop-relayer
description: Use when the user wants to fork or extend the reference Rust axum relayer — checkpoint backfill/resume so events aren't dropped on restart, custom storage backend (PostgreSQL, S3, …), new routes/handlers (e.g., a relayer-side Group Discovery endpoint backed by the groups index), sponsor-key strategy, operator policy (rate limiting, tenant scoping, abuse mitigation), or running it inside Nautilus. Trigger phrases - "extend the relayer", "fork the relayer", "add storage to relayer", "custom relayer handler", "PostgreSQL relayer", "relayer rate limiting", "checkpoint backfill", "checkpoint resume", "relayer missed events", "multi-tenant relayer", "relayer operator policy", "relayer abuse mitigation", "relayer group discovery endpoint", "expose memberships endpoint".
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

**A note on the abstraction:** the reference uses `Arc<dyn StorageAdapter>` for env-var-driven runtime swappability. A fork can pick the shape that fits its posture:

- **Trait object (status quo)** — keeps multiple backends behind one type at the cost of a v-table dispatch per call.
- **Generics** (`Relayer<S: StorageAdapter>`) — compile-time monomorphization; threads a generic param through anything that holds storage. (Note: `async_trait` already pays a boxed-future cost, so the dispatch win over `dyn` is smaller than for sync traits — pick generics for inlining and type-specific specialization, not micro-optimization.)
- **Concrete type** — drop the trait entirely; depend on `PostgresStorage` (or whatever) directly. Idiomatic when you have one backend forever and no test-double need that an in-memory shim can't cover. Trades swap-out for simplicity.

Don't carry the trait just because the reference does — pick the shape that matches your fork's posture.

### 2. Operator policy (rate limiting, tenant scoping, abuse mitigation)

Things the wire contract leaves to the operator:

- **Rate limiting** — `tower::limit::RateLimitLayer` on the router in `main.rs`, scoped per-IP, per-signer, or per-group as fits your hosting model.
- **Tenant scoping** — a multi-tenant relayer can constrain itself to a configured set of group IDs. Reject writes for group IDs outside that set before they hit the auth pipeline.
- **Abuse mitigation** — operator-side block of signer addresses or IPs as an emergency lever; the durable fix is removing the offending member from the group on-chain.
- **Pre-auth filtering** — payload size limits, malformed-header rejection, etc., to drop bad requests before signature verification.

Insert as axum middleware layers on the `Router` in `main.rs`, or as `from_fn_with_state` middleware before handler dispatch when shared config is needed.

Note: app-level access control (who can post, join gating, role permissions) is enforced on-chain by `sui_groups` and the messaging permission types — it belongs in your own Move package, not here. See [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md).

### 3. New endpoint

Add a handler under `src/handlers/`, register on the `Router` in `main.rs`. If the endpoint mutates state, run it through the same auth middleware so signatures are verified consistently.

A common motivating example: **expose a Group Discovery endpoint backed by the relayer's groups index** so clients can fetch a user's group memberships via a single REST call instead of running their own GraphQL queries (which is what `chat-app/` does today). `MembershipSyncService` already maintains the data; you only need a read handler over `MembershipStore` and a matching client caller.

Whichever endpoint you add, the SDK won't call it on its own. Pair this with [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md) to add the matching client method — either extend `HTTPRelayerTransport` (SDK-side fork) or implement a custom `RelayerTransport` that wraps the canonical methods plus your new one (keeps the SDK upgrade path clean).

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

### Pre-deploy safety checklist (for a fork going to mainnet)

- **`.env` secrets are out of git.** Confirm `git status` shows no `.env` staged; sponsor keys and admin keys never get committed even briefly.
- **Network alignment.** `GROUPS_PACKAGE_ID` matches the network the `SUI_RPC_URL` points to. Pointing a mainnet relayer at a testnet `GROUPS_PACKAGE_ID` (or vice versa) silently rejects every write — the membership store stays empty because the on-chain events it watches never fire.
- **Wire protocol unchanged, OR all consumers updated.** If you modified `handlers/messages/handlers.rs` types, `models/`, or `services/walrus_sync.rs`, every SDK client and the indexer that connect to your relayer have to be updated to the same wire format **before** your relayer goes live — otherwise existing canonical-SDK clients in the same groups stop being able to verify messages from your relayer's clients. This is the load-bearing constraint from the "Load-bearing surfaces" section above.
- **Sponsor-key custody.** If your fork sponsors transactions, the sponsor key controls real value and is a single point of failure. Use hardware-backed signing or a multisig, not a plaintext key file. Rotate periodically.
- **Capacity for the new network.** Mainnet checkpoint stream + Walrus testnet/mainnet are noisier than localnet — expect higher RAM and outbound bandwidth, and plan for the membership store growing without bound until you implement persistence (see "Checkpoint backfill / resume" above).
- **Don't run a forked relayer pointed at the canonical mainnet `sui_stack_messaging` package as a casual experiment.** Any messages your forked relayer accepts and archives become persistent state real users may try to read. Use testnet for forks-in-progress.

## Cross-links

- Run-local-only flow: [`spin-up-relayer`](../spin-up-relayer/SKILL.md).
- Full-stack with chat-app: [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md).
- Relayer protocol: `docs/sui-stack-messaging/Relayer.md`.
- Trust model (read before re-implementing auth): `docs/sui-stack-messaging/Security.md`.
- Relayer deep-dive: `relayer/README.md`.
