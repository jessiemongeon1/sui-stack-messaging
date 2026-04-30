---
name: develop-on-sui-stack-messaging
description: Use when starting a new project on Sui Stack Messaging, asking how to develop on or extend the messaging stack, or needing an overview of canonical packages vs reference implementations meant to be forked. Trigger phrases - "develop on sui-stack-messaging", "build on the messaging SDK", "where do I start", "extend the messaging stack", "what should I fork", "what's the architecture for development".
---

# Develop on Sui Stack Messaging

Top-level orientation for anyone building on this repo. Read this first, then jump to the child skill that matches your task.

## What's in this repo

```
groups-sdk/
├── move/                                       CANONICAL  Move smart contracts
│   └── packages/sui_stack_messaging/             ← consume as published; extend in your own package
├── ts-sdks/packages/sui-stack-messaging/        CANONICAL  TypeScript SDK
│   └── (consumed via @mysten/sui-stack-messaging on npm)
├── relayer/                                    REFERENCE  Rust axum relayer
├── walrus-discovery-indexer/                   REFERENCE  TS Walrus indexer
├── chat-app/                                   REFERENCE  Vite + React demo UI
├── publish/                                              Move publishing helper scripts
└── docs/sui-stack-messaging/                   Authoritative dev docs
```

## Canonical vs reference — the mental model

**Canonical** (`move/`, `ts-sdks/packages/sui-stack-messaging/`): consume; do not fork.

- Move contracts are already published to mainnet (`0xcbd2f4c25c7f799c45c0c9f221850178b711b2c89916c8e99038aa8ac609a62e`) and testnet (`0x047696be0e98f1b47a99727fecf2955cadb23c56f67c6b872b74e3ad59d51b46`). See `move/packages/sui_stack_messaging/Published.toml`.
- The SDK is on npm as `@mysten/sui-stack-messaging`.
- To extend behavior, write your **own** Move package that depends on `sui_stack_messaging` (see `extend-smart-contracts`) or implement custom transports/storage adapters via the SDK's interfaces.

**Reference** (`relayer/`, `walrus-discovery-indexer/`, `chat-app/`): fork as a starting point and modify freely.

- These are intentionally minimal and unopinionated. Real deployments will replace storage backends, auth strategies, observability, deployment topology.
- The SDK talks to the relayer through the `RelayerTransport` interface — you can implement your own transport in TS without touching this Rust code, or fork this Rust code.

**External canonical dep**: [`sui-groups`](https://github.com/MystenLabs/sui-groups) — the generic permissioned-groups library that messaging is built on. Treat it as published infrastructure; don't vendor it.

## Pick a child skill

| Goal                                                            | Skill                                                          |
| --------------------------------------------------------------- | -------------------------------------------------------------- |
| Run the reference relayer locally                               | [`spin-up-relayer`](../spin-up-relayer/SKILL.md)               |
| Run relayer + indexer + chat-app together end-to-end            | [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md)           |
| Fork-and-extend the Rust relayer (storage, auth, handlers)      | [`develop-relayer`](../develop-relayer/SKILL.md)               |
| Fork-and-extend the TS Walrus indexer (filters, storage, sinks) | [`develop-walrus-indexer`](../develop-walrus-indexer/SKILL.md) |
| Add custom Move modules (seal policies, join rules, gating)     | [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md) |

## What kinds of customization are expected

These are the typical extension points — surface area was designed for them:

1. **Custom relayer logic** — sponsor-key strategy, rate limiting, allowlists, persistence backend, observability. Fork `relayer/`.
2. **Custom archived-message recovery / Walrus blob discovery** — alternative archival-recovery flows or analytics over what the relayer wrote to Walrus. Fork `walrus-discovery-indexer/`. (Note: this is *not* user-facing "Group Discovery" — that's an SDK + Sui GraphQL concern; see `docs/sui-stack-messaging/GroupDiscovery.md`.)
3. **Custom Seal policies** — token-gated, subscription-based, or any application-specific access control. Add a Move module that implements `seal_approve_*` (see `move/packages/example_app/sources/custom_seal_policy.move`).
4. **Custom join rules** — paid joins, reputation gates, etc. (see `move/packages/example_app/sources/paid_join_rule.move`).
5. **Custom transports** — implement the `RelayerTransport` TS interface. Whether you also need to fork the relayer depends on the transport:
   - **In-process / queue / Nautilus-attested with HTTP semantics** — TS-only; no relayer fork.
   - **WebSocket / SSE / any non-HTTP wire format** — the reference relayer is HTTP-only (axum 0.7, no WS/SSE endpoints). You will need to fork it to expose the new transport server-side.
6. **Custom storage adapters** — implement the `StorageAdapter` TS interface for attachments to use S3/IPFS/etc. instead of Walrus.
7. **Custom recovery transports** — `RecoveryTransport` interface for alternative archive sources.

## Known gaps in the reference implementations

Two things you'll likely want to add early — they are not in the reference impls:

- **Checkpoint backfill / resume** — neither the relayer (`relayer/src/services/membership_sync.rs`) nor the indexer (`walrus-discovery-indexer/src/checkpoint-listener.ts`) persists or resumes from a checkpoint cursor. Both subscribe to the live Sui gRPC checkpoint stream from "now" on each start, so events emitted in any restart/downtime gap are dropped. The canonical fix walks historical checkpoints via gRPC `LedgerService.GetCheckpoint(sequence_number)` between the persisted cursor and the live tip, persisting a `(sequence, last_tx_digest)` pair so resume can skip already-processed transactions in the boundary checkpoint. See [`develop-relayer`](../develop-relayer/SKILL.md) and [`develop-walrus-indexer`](../develop-walrus-indexer/SKILL.md) for the per-service detail and the TypeScript reference implementation.
- **Persistent storage** — both services default to in-memory stores. State is lost on restart. Pair with checkpoint resume above so a new instance can rebuild state.

## What is NOT a customization point

- **The wire format / on-chain object layout** of `sui_stack_messaging`. Don't redefine these structs in a fork.
- **Encryption envelope format**. Implementing your own encryption layer breaks cross-client interoperability.
- **Permission types declared in `messaging.move`** (`MessagingSender`, `MessagingReader`, `MessagingEditor`, `MessagingDeleter`, `MetadataAdmin`, `SuiNsAdmin`). Add new permission types in your own package; do not redefine these.

## Authoritative docs to read before extending

- `docs/sui-stack-messaging/Setup.md` — client instantiation reference.
- `docs/sui-stack-messaging/Extending.md` — custom Seal policies, transports, storage adapters, recovery transports.
- `docs/sui-stack-messaging/Security.md` — trust boundaries; read before implementing custom auth or transport.
- `docs/sui-stack-messaging/Relayer.md` — protocol the relayer implements; required reading before forking.
- `move/design_docs/REQUIREMENTS.md` — Move package architecture.

## Runnable usage examples

The SDK ships an integration test suite at `ts-sdks/packages/sui-stack-messaging/test/integration/localnet/` that runs against a localnet docker stack and doubles as a living examples directory — full Move + SDK + relayer flows, end-to-end.

| File                                                  | Shows                                                                               |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `setup-and-config.test.ts`                            | Vanilla messaging client wiring (good baseline).                                    |
| `flows.test.ts`                                       | Common message-flow patterns.                                                       |
| `custom-seal-policy.test.ts`                          | Wiring a custom `SealPolicy` end-to-end (maps to extend-smart-contracts Pattern 1). |
| `paid-join-rule.test.ts`                              | Actor object pattern end-to-end (maps to extend-smart-contracts Pattern 2).         |
| `example-apps-setup-and-config.test.ts`               | Full client extension chain with the `example_app` Move package.                    |
| `view.test.ts`, `metadata.test.ts`, `archive.test.ts` | Read-side queries, group metadata, archival/recovery.                               |

Run with `pnpm test:integration` from `ts-sdks/packages/sui-stack-messaging/`. See `docs/sui-stack-messaging/Testing.md` for the localnet docker setup.

The unit suite at `ts-sdks/packages/sui-stack-messaging/test/unit/` is more granular — useful for understanding individual components (`seal-policy.test.ts`, `envelope-encryption.test.ts`, `dek-manager.test.ts`, `attachments-manager.test.ts`, `verification.test.ts`, `http-transport.test.ts`).

## Tooling baseline

- Node: pnpm `>=10.17.0` (see `ts-sdks/package.json`).
- Rust: stable toolchain with `clippy` + `rustfmt` (see `relayer/rust-toolchain.toml`).
- Sui CLI: required for Move build/publish. Toolchain pinned at 1.68.1 in `Published.toml`.
- Move edition: 2024.

## Repo invariants

- `ts-sdks/packages/sui-stack-messaging/src/contracts/sui_stack_messaging/` is auto-generated. Never edit by hand; regenerate with `pnpm codegen` after Move changes.
- `Published.toml` is committed and is the source of truth for deployed addresses. Do not delete it.
- The SDK depends on `@mysten/sui-groups` and `@mysten/seal` as peer deps — version bumps are coordinated via `pnpm changeset` (see `ts-sdks/RELEASING.md`).
