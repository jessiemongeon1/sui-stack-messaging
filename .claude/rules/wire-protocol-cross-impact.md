---
paths:
  - "ts-sdks/packages/sui-stack-messaging/src/relayer/**"
  - "ts-sdks/packages/sui-stack-messaging/src/verification.ts"
  - "ts-sdks/packages/sui-stack-messaging/src/recovery/**"
  - "relayer/src/handlers/**"
  - "relayer/src/models/**"
  - "relayer/src/services/walrus_sync.rs"
  - "walrus-discovery-indexer/src/event-parser.ts"
  - "walrus-discovery-indexer/src/blob-inspector.ts"
  - "walrus-discovery-indexer/src/discovery-store.ts"
  - "walrus-discovery-indexer/src/api.ts"
  - "walrus-discovery-indexer/src/types.ts"
  - "docs/sui-stack-messaging/Relayer.md"
---

# Wire-protocol changes have cross-component impact — coordinate all three sides

The files matched by this rule define **wire contracts shared across three independently-deployed components**: the TypeScript SDK, the Rust relayer, and the TypeScript walrus-discovery-indexer. A change in any one place that breaks the contract silently breaks the other two — often in ways that don't surface until a real cross-component interaction (auth failure, decode error, missing field).

## The three load-bearing contracts

1. **HTTP request/response shape** between SDK ↔ relayer.
   - SDK side: `ts-sdks/packages/sui-stack-messaging/src/relayer/http-transport.ts`, `transport.ts`, `types.ts`.
   - Relayer side: `relayer/src/handlers/messages/*`, `relayer/src/models/*`.
   - Authoritative doc: `docs/sui-stack-messaging/Relayer.md`.

2. **Canonical-message format for signature verification** (security-critical).
   - SDK side: `ts-sdks/packages/sui-stack-messaging/src/verification.ts` (`buildCanonicalMessage` / `verifyMessageSender`).
   - Relayer side: the same canonical format is reconstructed and verified in `relayer/src/auth/` and in the message models in `relayer/src/models/`.
   - Authoritative doc: `docs/sui-stack-messaging/Security.md`.
   - **Drift here means signatures the SDK produces won't verify on the relayer (silent auth failures) or vice versa.** Other group members also re-verify with this same format — diverging means *every other client* in the same group stops being able to verify your messages.

3. **Walrus archive format** between relayer (writer) ↔ indexer (reader) ↔ SDK `RecoveryTransport` (consumer of the indexer's output).
   - Writer: `relayer/src/services/walrus_sync.rs`.
   - Reader: `walrus-discovery-indexer/src/event-parser.ts`, `blob-inspector.ts`.
   - Consumer: `ts-sdks/packages/sui-stack-messaging/src/recovery/` and the indexer's REST API in `walrus-discovery-indexer/src/api.ts`.
   - Existing archives stop being readable on schema flips — treat as a versioned contract: bump a format version and support both during transition.

## What to do when you must change a contract

- **Update all three sides in the same PR.** Coordinating across two PRs lands you in a window where the components are mutually broken.
- **Update `docs/sui-stack-messaging/Relayer.md` in the same PR** — it's the protocol spec downstream forks rely on.
- **Prefer additive over breaking:** a new endpoint, a new permission type, a new archive variant. Keep old shape supported for at least one release.
- **Bump a format version field** and accept both old and new during the transition; remove the old branch in a later release.
- **If the work is exploratory:** start with the smallest change to all three sides that round-trips a single test case end-to-end (SDK → relayer → Walrus → indexer → SDK recovery), then expand.

## Verification checklist before declaring done

- [ ] `cargo test` passes in `relayer/` (signature verification still works on round-trip).
- [ ] `pnpm test:unit` passes in `ts-sdks/packages/sui-stack-messaging/` (verification.ts unit tests + http-transport.ts unit tests).
- [ ] `pnpm test:integration` runs the localnet-docker stack end-to-end at least once with the new contract.
- [ ] `pnpm test` passes in `walrus-discovery-indexer/`.
- [ ] `docs/sui-stack-messaging/Relayer.md` reflects the new shape.
