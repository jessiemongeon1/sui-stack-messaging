---
name: configure-custom-relayer-transport
description: Use when a developer wants to bring their own relayer (or sponsor / gas model) without forking the reference Rust relayer — they implement the RelayerTransport TypeScript interface in their app, hand it to the SDK via `relayer.transport`, and the SDK uses it in place of HTTPRelayerTransport. Covers when to implement vs fork, the seven methods of the interface, sponsor-key trade-offs, and wire-protocol compatibility if you still want to interop with canonical relayers. Trigger phrases - "custom RelayerTransport", "don't use the reference relayer", "BYO relayer", "sponsor my own gas", "implement RelayerTransport", "WebSocket relayer", "in-process relayer", "Nautilus-attested relayer".
---

# Custom RelayerTransport — bring your own backend

The SDK's default path is `HTTPRelayerTransport` (`ts-sdks/packages/sui-stack-messaging/src/relayer/http-transport.ts`), which speaks the HTTP wire protocol the reference Rust relayer implements. If you want a different backend — a different sponsor model, a different wire protocol (WebSocket, SSE, in-process queue), a Nautilus-attested execution environment, or simply your own HTTP service that you own end-to-end — you implement the `RelayerTransport` interface in TypeScript and hand it to the SDK via `relayer.transport`.

**You do not need to fork the reference Rust relayer for this path.** Forking is a separate option ([`develop-relayer`](../develop-relayer/SKILL.md)) for when you specifically want to extend the canonical HTTP relayer (custom storage, custom auth middleware, etc.).

## When to use this skill vs other paths

| Situation | Path |
|---|---|
| You want a non-HTTP-polling wire format (SSE, WebSocket, long-poll, hybrid) | **This skill** (implement `RelayerTransport` in TS) + a backend that speaks it |
| You want a different HTTP service that you control end-to-end | **This skill** (or fork the Rust relayer — your call) |
| You want an in-process / queue backend | **This skill** |
| You want to run the reference relayer inside a TEE (Nautilus) | Use the community Nitro-enclave template: see [`docs/sui-stack-messaging/CommunityContributed.md`](../../../docs/sui-stack-messaging/CommunityContributed.md) — this is a deployment topology, not a wire-protocol change, so no custom transport is needed |
| You want to fix gaps in the reference relayer itself (persistence, backfilling, zkLogin auth) | [`develop-relayer`](../develop-relayer/SKILL.md) — these are server-side concerns |
| You want to add storage / auth / observability / sponsored-txs to the canonical Rust relayer | [`develop-relayer`](../develop-relayer/SKILL.md) |
| You want to swap how attachments are stored | [`configure-walrus-storage-via-sdk`](../configure-walrus-storage-via-sdk/SKILL.md) |
| You want to just run the reference relayer for dev | [`spin-up-relayer`](../spin-up-relayer/SKILL.md) |

## The `RelayerTransport` interface

Seven methods. The interface is in `ts-sdks/packages/sui-stack-messaging/src/relayer/transport.ts`; the param/result types are in `ts-sdks/packages/sui-stack-messaging/src/relayer/types.ts`.

```ts
interface RelayerTransport {
  sendMessage(params: SendMessageParams): Promise<SendMessageResult>;
  fetchMessages(params: FetchMessagesParams): Promise<FetchMessagesResult>;
  fetchMessage(params: FetchMessageParams): Promise<RelayerMessage>;
  updateMessage(params: UpdateMessageParams): Promise<void>;
  deleteMessage(params: DeleteMessageParams): Promise<void>;
  subscribe(params: SubscribeParams): AsyncIterable<RelayerMessage>;
  disconnect(): void;
}
```

Important shape details (from `types.ts`):

- **`RelayerMessage` carries `signature` + `publicKey` per message.** These are hex-encoded; the SDK uses them to independently re-verify sender authenticity via `verifyMessageSender` (`ts-sdks/packages/sui-stack-messaging/src/verification.ts`) — your transport must preserve them losslessly. *Other group members re-verify the same way; if you drop or rewrite these fields, no canonical-SDK client in the group will be able to verify your messages.*
- **`encryptedText` and `nonce` are `Uint8Array`** — your transport never sees plaintext. Don't try to inspect or transform them.
- **`order` is monotonically increasing per group.** `subscribe(afterOrder)` lets a client resume from a known cursor. Implementations must deliver every message with `order > afterOrder`, in order.
- **`syncStatus` and `quiltPatchId` are optional.** Only relevant if your backend syncs to Walrus. Leave undefined otherwise.
- **`messageSignature` (param) is hex-encoded.** Persist it; it's what `signature` (result) needs to be on subsequent fetches.

## Wire-protocol compatibility

This skill covers two distinct scenarios:

### Scenario A — fully custom backend, no canonical interop required

You don't care about wire-protocol compatibility with the canonical Rust relayer; your `RelayerTransport` is the only consumer. **You're free to use any wire format you want** — gRPC, WebSocket, in-process function calls, whatever. Just preserve the field semantics above (especially `signature` / `publicKey` round-tripping).

### Scenario B — your transport talks to multiple relayer types, including canonical

If your `RelayerTransport` needs to also work against the canonical HTTP relayer (e.g., your dapp supports both your custom backend and the reference one), then your custom HTTP relayer must speak the **exact** wire protocol documented in [`docs/sui-stack-messaging/Relayer.md`](../../../docs/sui-stack-messaging/Relayer.md), and your transport's HTTP client logic must match `HTTPRelayerTransport` byte-for-byte for the canonical endpoints. The path-scoped rule [`wire-protocol-cross-impact`](../../rules/wire-protocol-cross-impact.md) applies — if you change the protocol, the canonical SDK + reference relayer + indexer all need updates too.

For most Builders, Scenario A is the right framing.

## Wire it into the SDK

```ts
import type { RelayerTransport } from '@mysten/sui-stack-messaging';
import { createSuiStackMessagingClient } from '@mysten/sui-stack-messaging';

class MyTransport implements RelayerTransport {
  // ... seven method implementations
}

// Factory name is `createSuiStackMessagingClient` (per src/factory.ts); some docs
// show an older `createMessagingGroupsClient` — trust the source export.
const client = createSuiStackMessagingClient(baseClient, {
  // ...seal, encryption
  relayer: {
    transport: new MyTransport({ /* your config */ }),
    // note: NO relayerUrl when transport is supplied — types enforce mutual exclusion
  },
});
```

The config types (`types.ts`) explicitly forbid passing both `relayerUrl` and `transport` — they're mutually exclusive (`RelayerHTTPConfig` vs `RelayerCustomTransportConfig`).

## Why Builders customize the transport (common drivers)

These are the typical motivations for either implementing a custom `RelayerTransport` in TS, forking the Rust relayer, or both. They're roughly of equal weight — but **#1 is effectively a prerequisite for any production deployment**; the rest are choices on top of it.

1. **Fix implementation gaps in the reference relayer** — persistence (the reference uses in-memory storage), checkpoint backfill / resume on restart, zkLogin auth-scheme support, etc. These are **server-side**; address them via [`develop-relayer`](../develop-relayer/SKILL.md). The TS `RelayerTransport` shape doesn't change.
2. **Non-HTTP-polling wire formats** — SSE, WebSockets, long-polling, hybrids for more complex client topologies. Requires both a backend that speaks the new wire format *and* a `RelayerTransport` implementation here that talks to it. The reference Rust relayer is HTTP-only (axum, no WS/SSE endpoints today) — non-HTTP paths typically need a relayer fork *plus* this skill.
3. **More complex software architectures for scalability** — multi-region deployments, message queues between ingest and worker tiers, read-replica fan-out, edge caches in front of fetch endpoints. The seven-method surface stays the same; what changes is what your transport speaks to (and what the backend looks like on the other side).
4. **Sponsored transactions / gas models** — out of scope for this skill. The reference Rust relayer does **not** sponsor on-chain txs; clients sign their own. If you need sponsorship, defer to the canonical references: [Sui Sponsored Transactions](https://docs.sui.io/guides/developer/sui-101/sponsor-txn) for the protocol-level mechanics and [Mysten Enoki](https://enoki.mystenlabs.com/) for a managed sponsor / gas-station / zkLogin offering. Wiring either into your relayer or transport is your responsibility.
5. **Additional features / endpoints** — anything the canonical wire protocol doesn't cover (custom moderation hooks, app-specific metadata, server-driven push notifications, …). Either extend the wire protocol on a forked Rust relayer with a matching transport, or build a side-channel API that lives outside `RelayerTransport`.

## Verification

> **Safety:** steps (2) and (4) below send real messages and (depending on the SDK methods you call to set up the group) may mint on-chain group objects. Run them with a **dev signer on testnet or localnet**, against a **throwaway group** you created for this test only. Never verify a custom transport against a production group or a wallet holding real value. Prefer the localnet integration test path (step 3) for anything close to a regression suite — localnet is hermetic and disposable.

1. **Implement against the interface in isolation** (no SDK wiring yet) — write a tiny unit test that calls every method on your transport and asserts the right side effects (HTTP fetch / queue publish / etc.). Non-mutating. Much faster to iterate on than full-stack tests.
2. **Round-trip `signature` + `publicKey`** (**mutating** — uses a dev group): send a message via the SDK against a throwaway dev group, fetch it back, run `verifyMessageSender` from [`ts-sdks/packages/sui-stack-messaging/src/verification.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/verification.ts) — it should return true. If it doesn't, you're dropping or transforming the per-message signature fields.
3. **End-to-end via integration tests** (localnet, hermetic): copy [`ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts`](../../../ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts) and swap the transport. A pass means the SDK can send + fetch + decrypt against your transport. **Preferred** for regression suites — localnet state is thrown away between runs.
4. **Subscribe cursor behavior** (**mutating** — disposable test group): call `subscribe({ afterOrder: N })`, send 3 messages, assert they arrive in the order the backend returned them. Then disconnect, send more, re-subscribe from the latest seen `order`, assert no gaps and no duplicates. This validates your *cursor handling* (exclusive `afterOrder`) and pass-through fidelity — not order assignment, which is the backend's job. Run against a fresh dev group; the test sends real messages.

## Common pitfalls

These are transport-side mistakes. `order` assignment, monotonicity, and storage-layer consistency are **backend** concerns — your transport just relays what the backend returns. Backend-side ordering pitfalls (e.g. when you fork the Rust relayer) live in [`develop-relayer`](../develop-relayer/SKILL.md), not here.

- **Mutating or reordering fetched messages before handing them to the SDK.** Whatever sequence the backend returned in `fetchMessages` / `subscribe` is the sequence the SDK expects to consume (the backend assigns `order` on ingress — see `relayer/src/models/message.rs:120`). Don't re-sort, dedupe, or skip on the transport side; pass through.
- **Losing or rewriting `signature` / `publicKey` on `RelayerMessage`.** The SDK uses both for independent sender verification (see [`verification.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/verification.ts)); if either is missing or altered, `senderVerified` silently becomes `false` for every consumer in the group — including other group members running the canonical SDK.
- **Losing other fields on round-trip.** `attachments`, `syncStatus`, `quiltPatchId`, `isEdited`, `isDeleted`, `createdAt`, `updatedAt` all need to survive `fetchMessages` → `fetchMessage` → `updateMessage` round-trips. Field drops here surface as cryptic SDK errors elsewhere.
- **Mishandling pagination cursors.** `fetchMessages` takes `afterOrder` / `beforeOrder` (exclusive); `subscribe` takes `afterOrder` (exclusive — "Only messages with order > afterOrder are delivered" per [`types.ts:89`](../../../ts-sdks/packages/sui-stack-messaging/src/relayer/types.ts)). Off-by-one or inclusive handling causes silent duplicates or gaps.
- **Throwing generic `Error`s instead of `RelayerTransportError`.** The SDK inspects `error.status` and `error.code` to surface useful messages. Use `new RelayerTransportError(message, status, code?)` from [`relayer/types.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/relayer/types.ts) (`code` is optional; `status` is the HTTP-style code such as 401 / 404 / 405).

## Where this fits

- Bigger Builder integration story: [`integrate-sui-stack-messaging`](../integrate-sui-stack-messaging/SKILL.md).
- Authoritative wire protocol (relevant if you go Scenario B): [`docs/sui-stack-messaging/Relayer.md`](../../../docs/sui-stack-messaging/Relayer.md).
- SDK reference: [`docs/sui-stack-messaging/Extending.md`](../../../docs/sui-stack-messaging/Extending.md) § "Custom transport".
- Fork the reference relayer instead: [`develop-relayer`](../develop-relayer/SKILL.md).
- The path-scoped wire-protocol rule: [`.claude/rules/wire-protocol-cross-impact.md`](../../rules/wire-protocol-cross-impact.md).
