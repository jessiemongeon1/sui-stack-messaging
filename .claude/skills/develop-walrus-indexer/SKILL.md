---
name: develop-walrus-indexer
description: Use when the user wants to fork or extend the reference TypeScript walrus-discovery-indexer — custom event filters, persistent storage backend (PostgreSQL/MongoDB instead of in-memory), output sinks (webhooks, queues), custom blob discovery rules, or BCS event parsing. Trigger phrases - "extend the walrus indexer", "fork the indexer", "custom event filter", "indexer storage backend", "blob discovery rules", "indexer webhook", "walrus discovery custom".
---

# Develop the Walrus discovery indexer

The reference indexer at `walrus-discovery-indexer/` is a TypeScript service that subscribes to Sui checkpoints, inspects certified Walrus blobs, and exposes a REST API for client-side **archived-message recovery**. Fork it freely.

> **Terminology — read this once.** "Discovery" in `walrus-discovery-indexer` means **discovering archived messages on Walrus** (i.e., locating Walrus blobs/quilts the relayer wrote during archival, so SDK clients can rebuild message history). It is **not** the same as the user-facing **"Group Discovery"** feature documented in `docs/sui-stack-messaging/GroupDiscovery.md`, which is about discovering `PermissionedGroup<Messaging>` objects the connected user is a member of — a separate concern handled by the SDK + Sui GraphQL queries (the chat-app's `chat-app/src/hooks/useGroupDiscovery.ts` is the reference). This service has nothing to do with that.

## Mental model

You're forking a reference — not modifying a library. Don't worry about preserving an upstream API. Do preserve:

- **The blob inspection contract**: the SDK's `RecoveryTransport` queries this indexer to reconstruct message history. If you change response shapes, update the SDK side too.
- **The 3-tier filter pipeline** (sender → event type → tag-based) is load-bearing for keeping mainnet noise manageable. Add filters; don't remove them unless you have a reason.

Everything else — storage backend, additional sinks, custom analytics — is yours.

**Language and runtime are a fork choice, not a contract.** This reference is TypeScript because it pairs naturally with the SDK's `RecoveryTransport` consumer, but nothing about the indexer's role requires TS. If your stack is Rust, reimplement the same logical pieces — Sui gRPC checkpoint subscription, Walrus blob/quilt inspection, persistent cursor + backfill, REST endpoints matching the response shapes the SDK consumes — in your language of choice. The rest of this skill assumes you're forking the TS code (the per-file extension points are TS-specific), but the architecture and contracts apply identically to any reimplementation.

## Run + test loop

```bash
cd walrus-discovery-indexer
pnpm install

pnpm dev          # tsx watch src/index.ts (live reload)
pnpm build        # tsc -> dist/
pnpm start        # node dist/index.js
pnpm test         # vitest run (test/**/*.test.ts)
```

No linter is wired in; format and typecheck are implicit via `tsc`. If you add ESLint/Prettier, follow `ts-sdks/` conventions.

## Source layout

```
src/
├── index.ts                       wires config + clients + REST + checkpoint listener
├── checkpoint-listener.ts         Sui gRPC checkpoint subscription, sender filter
├── blob-inspector.ts              Walrus tag-based blob filtering
├── discovery-store.ts             DiscoveryStore interface + InMemoryDiscoveryStore
├── event-parser.ts                BCS struct parsing for BlobCertified events
└── api.ts                         Express REST endpoints
test/                              vitest specs
```

## Common extensions

### 0. Checkpoint backfill / resume (usually first)

The reference indexer does not backfill. `startCheckpointListener` (`src/checkpoint-listener.ts`) calls `grpcClient.subscriptionService.subscribeCheckpoints` with no starting cursor on every connect — including after auto-reconnect. `store.setLastCheckpoint(checkpointSeq)` records the latest seen checkpoint but is **not** read back as a resume cursor. So events emitted while the indexer was down are not discovered.

The canonical pattern walks historical checkpoints via the same gRPC service as the live tail. Shape:

1. **Persist a cursor pair** `(checkpoint_sequence, last_processed_tx_digest)` in your `DiscoveryStore`. The tx digest lets you resume mid-checkpoint without re-emitting blobs you already inspected.
2. **On startup**, read the persisted cursor, then walk forward one checkpoint at a time via `grpcClient.ledgerService.getCheckpoint({ checkpointId: { sequenceNumber: seq }, readMask })`. Skip transactions in the boundary checkpoint until you pass `last_processed_tx_digest`; process every transaction normally in subsequent checkpoints. Persist the cursor after each one.
3. **Then** subscribe to `subscribeCheckpoints` for the live tail.
4. **On reconnect**, repeat step 2 from the persisted cursor to the new tip before re-joining the live stream.
5. **Per-blob inspection retries** — if `inspectBlob` fails (network blip), the in-memory current code drops it; with persistent storage you can mark "seen, not yet inspected" and retry later.

Without backfill, a fresh indexer instance returns empty results until new blobs are certified. Pair this with a persistent `DiscoveryStore` (next section) — backfill is what makes the persisted store actually useful.

### 1. Persistent storage (PostgreSQL / Mongo / Redis)

`src/discovery-store.ts` declares the `DiscoveryStore` interface. Replace `InMemoryDiscoveryStore` with your impl and pass it into `createApp()` and the listener.

```ts
// src/postgres-store.ts
import { DiscoveryStore } from "./discovery-store.js";
export class PostgresDiscoveryStore implements DiscoveryStore {
  async upsert(blob) {
    /* SQL */
  }
  async query(filter) {
    /* SQL */
  }
}
```

Wire in `src/index.ts`:

```ts
const store =
  process.env.STORE === "postgres"
    ? new PostgresDiscoveryStore({ url: process.env.DATABASE_URL })
    : new InMemoryDiscoveryStore();
```

### 2. Custom event filter

Two layers exist. Pick the right one:

- **Sender filter (tier 1)** — `src/checkpoint-listener.ts`. Cheap; runs before any RPC. Add allowlist/blocklist of sender addresses here.
- **Tag filter (tier 3)** — `src/blob-inspector.ts`. Runs after Walrus quilt index fetch. Add custom tag rules (e.g. only blobs with `app=myapp`).

### 3. Output sink (webhook / queue / file)

`src/api.ts` is the REST output today. To push instead of poll, add a sink that fires on `store.upsert()`:

```ts
// src/sinks/webhook.ts
export class WebhookSink {
  constructor(private url: string) {}
  async emit(blob) {
    await fetch(this.url, { method: "POST", body: JSON.stringify(blob) });
  }
}
```

Then call `sink.emit(blob)` after a successful insert in the listener.

### 4. New event type / BCS struct

`src/event-parser.ts` defines `BlobCertifiedBcs`. To parse other Walrus events, add a new BCS struct, register the event type filter in `checkpoint-listener.ts`, and dispatch parsing.

### 5. Custom REST endpoint

Extend `createApp()` in `src/api.ts`. Keep response shapes documented — the SDK's recovery transport may consume them.

## Config

From `walrus-discovery-indexer/.env.example`:

| Var                            | Required | Default   | Purpose                                                             |
| ------------------------------ | -------- | --------- | ------------------------------------------------------------------- |
| `NETWORK`                      | yes      | `testnet` | `testnet` \| `mainnet` — picks Sui gRPC endpoint                    |
| `WALRUS_PUBLISHER_SUI_ADDRESS` | no       | (none)    | Tier-1 sender filter; without it, every certified blob is inspected |
| `PORT`                         | no       | `3001`    | REST port                                                           |

Add your own (e.g., `DATABASE_URL`, `WEBHOOK_URL`) and document them next to these.

## What NOT to change

- The blob/quilt discovery format — that's defined by the relayer's archival code (`relayer/src/services/walrus_sync.rs`). The indexer reads what the relayer writes; if you change the format, change both.
- The semantics of REST responses consumed by the SDK's `RecoveryTransport`.

## Tests

`vitest` is wired (`vitest.config.ts`). The example spec is `test/event-parser.test.ts`. Mirror that pattern: pure-function tests for parsing/filtering, integration tests can spin up a fake gRPC stream.

## Deployment notes

- `Dockerfile` exists — `docker build -t walrus-indexer .`.
- For mainnet, set `WALRUS_PUBLISHER_SUI_ADDRESS` to your relayer's address; **without it, you'll inspect the entire mainnet blob stream** — that's a lot of RPC + Walrus aggregator load on shared infrastructure, and it'll fill your in-memory store unboundedly. Always set the sender filter when pointing this at mainnet (or any shared/production network).

### Pre-deploy safety checklist (for a fork going to mainnet)

- **Network alignment.** `NETWORK=mainnet` matches the network of the relayer whose `WALRUS_PUBLISHER_SUI_ADDRESS` you're filtering on. Mixing networks gets you zero results silently.
- **Persistent storage in place.** The default `InMemoryDiscoveryStore` loses everything on restart. For any deployment that survives a process restart, swap to a persistent backend (see "Persistent storage" above) **and** implement checkpoint backfill (see "Checkpoint backfill / resume" above) — without backfill, a fresh persistent store starts empty just like the in-memory one and silently drops everything that landed before startup.
- **Wire protocol unchanged, OR coordinated with the relayer + SDK.** If you changed `event-parser.ts`, `blob-inspector.ts`, or REST response shapes in `api.ts`, both the relayer's `walrus_sync.rs` (writer) and the SDK's `RecoveryTransport` (consumer) need to know. See the path-scoped rule [`.claude/rules/wire-protocol-cross-impact.md`](../../rules/wire-protocol-cross-impact.md).
- **Resource budget.** Mainnet checkpoint subscription pulls every checkpoint; even with the sender filter applied at the indexer (tier 1), the gRPC subscription itself does not pre-filter. Plan for sustained outbound bandwidth and CPU.
- **Don't run a hobby indexer against canonical mainnet infrastructure** if the only consumer is you — use testnet. An indexer that crashes mid-stream is harmless on testnet; on mainnet it costs the operator real RPC budget and may rate-limit you out of the shared endpoint.

## Cross-links

- Indexer in the full local stack: [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md).
- Indexer reference doc: `walrus-discovery-indexer/README.md` and `walrus-discovery-indexer/docs/README.md`.
- SDK recovery transport (the consumer of this indexer): `docs/sui-stack-messaging/ArchiveRecovery.md`.
