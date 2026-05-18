---
name: configure-walrus-storage-via-sdk
description: Use when a developer wants to implement a custom StorageAdapter that talks to Walrus through the @mysten/walrus TypeScript SDK directly — instead of the default WalrusHttpStorageAdapter which depends on public publisher and aggregator HTTP endpoints. Useful for production setups that need to avoid the public Walrus infrastructure (no third-party dependency, no rate-limit risk, full control over how store/read txs are funded and submitted). Trigger phrases - "custom Walrus storage", "don't want public publishers/aggregators", "use @mysten/walrus directly", "programmatic Walrus", "Walrus SDK adapter", "implement StorageAdapter with walrus SDK", "skip publisher", "client-side Walrus".
---

# Custom Walrus storage via the @mysten/walrus SDK

The canonical attachments path uses `WalrusHttpStorageAdapter` (`ts-sdks/packages/sui-stack-messaging/src/storage/walrus-http-storage-adapter.ts`), which talks to a Walrus **publisher** (HTTP `PUT /v1/quilts`) and **aggregator** (HTTP `GET /v1/blobs/by-quilt-patch-id/{id}`). That depends on third-party publisher/aggregator infrastructure being reachable, healthy, and not rate-limiting you.

This skill covers the alternative: implement your own `StorageAdapter` that uses the `@mysten/walrus` TypeScript SDK to perform the equivalent operations **programmatically and client-side** — signing and submitting Walrus store/read transactions yourself rather than POSTing to a publisher.

## Scope decision — staying within Walrus

Walrus is the canonical storage layer for this SDK. This skill does **not** recommend replacing Walrus with S3, IPFS, or another non-Walrus backend. The choice is purely about *how* you talk to Walrus:

| Path | Pros | Cons |
|---|---|---|
| **Default — `WalrusHttpStorageAdapter`** | Zero setup; works in the browser; no per-upload tx fee paid by user | Depends on a third-party publisher/aggregator being up + uncapped |
| **This skill — `@mysten/walrus` SDK** | No third-party HTTP dependency; you control every tx | You sign and submit store/read txs yourself; you need WAL tokens or a sponsor |

If you genuinely need a non-Walrus backend, the `StorageAdapter` interface is backend-agnostic (`storage-adapter.ts` deliberately leaves `metadata` as `unknown` and the `name → id` mapping pluggable), but that's a different skill and not currently a recommended path.

## The `StorageAdapter` interface

```ts
// ts-sdks/packages/sui-stack-messaging/src/storage/storage-adapter.ts

interface StorageEntry { name: string; data: Uint8Array; }

interface StorageUploadResult {
  ids: string[];          // one per entry, in input order; used by download()
  metadata?: unknown;     // adapter-specific; opaquely persisted by the SDK
}

interface StorageAdapter {
  upload(entries: StorageEntry[]): Promise<StorageUploadResult>;
  download(id: string): Promise<Uint8Array>;
  delete?(ids: string[]): Promise<void>;  // optional
}
```

Three contracts your implementation must honor:

1. **Encryption-agnostic.** Entries arrive already encrypted. Do not re-encrypt; do not inspect.
2. **`ids[]` is positionally aligned with the input `entries[]`.** Consumers index into it by position.
3. **`metadata` is opaque to the consumer.** Whatever you return is persisted by `AttachmentsManager` and may be passed back later (e.g., for epoch-extension or deletion txs). Define a typed shape and document it.

## Implementation shape

```ts
import type {
  StorageAdapter,
  StorageEntry,
  StorageUploadResult,
} from '@mysten/sui-stack-messaging';
import { WalrusClient } from '@mysten/walrus';
import type { Signer } from '@mysten/sui/cryptography';
import type { SuiClient } from '@mysten/sui/client';

export interface WalrusSdkAdapterConfig {
  walrusClient: WalrusClient;           // configured for testnet/mainnet
  signer: Signer;                       // signs store + delete txs (needs WAL + SUI for gas)
  epochs: number;                       // storage duration ahead of current epoch
}

export interface WalrusSdkUploadMetadata {
  // Mirror the shape WalrusHttpStorageAdapter returns so downstream code
  // can treat both adapters interchangeably for on-chain follow-ups.
  blobObjectId: string;
  blobId: string;
  startEpoch: number;
  endEpoch: number;
  cost: number;
  deletable: boolean;
}

export class WalrusSdkStorageAdapter implements StorageAdapter {
  constructor(private readonly cfg: WalrusSdkAdapterConfig) {}

  async upload(entries: StorageEntry[]): Promise<StorageUploadResult> {
    // 1. Combine the entries into a quilt (the @mysten/walrus SDK exposes a
    //    quilt builder; the input shape mirrors `entries` — each becomes a
    //    quilt patch identified by `name`).
    // 2. Build + sign + submit the Walrus store transaction with `this.cfg.signer`.
    // 3. Read the certified quilt patch IDs from the tx result.
    // 4. Return them positionally aligned with `entries`, plus a metadata
    //    object matching WalrusSdkUploadMetadata.
    // See @mysten/walrus README for the exact builder/submit calls.
  }

  async download(id: string): Promise<Uint8Array> {
    // Call the @mysten/walrus SDK's quilt-patch read by ID. No tx signing
    // required for reads — they're permissionless.
  }

  async delete(ids: string[]): Promise<void> {
    // Optional. Build + sign + submit the Walrus delete tx if the blob is
    // `deletable: true`. Skip otherwise (don't throw — the interface contract
    // allows this method to be absent or no-op for non-deletable blobs).
  }
}
```

The exact `@mysten/walrus` API calls (quilt builder, store/read methods, types) live in that SDK's own documentation. This skill is about **how to wrap them in the `StorageAdapter` interface**, not how to use Walrus itself.

For Walrus-side concepts (quilts, patches, epochs, certification flow) defer to the [Walrus docs](https://docs.wal.app/) and the `@mysten/walrus` README.

## Wire it into the messaging client

Pass an instance under `attachments.storageAdapter`:

```ts
// Factory name is `createSuiStackMessagingClient` per src/factory.ts; some docs show
// an older name (`createMessagingGroupsClient`) — trust source.
const client = createSuiStackMessagingClient(baseClient, {
  // ...seal, encryption, relayer
  attachments: {
    storageAdapter: new WalrusSdkStorageAdapter({
      walrusClient: myWalrusClient,
      signer: myAttachmentUploadSigner,
      epochs: 5,
    }),
  },
});
```

(`WalrusSdkStorageAdapter` and `WalrusSdkUploadMetadata` above are *your* implementation's names — choose what you like. The canonical reference is `WalrusHttpStorageAdapter` / `WalrusUploadMetadata` in [`storage/walrus-http-storage-adapter.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/storage/walrus-http-storage-adapter.ts).)

Without `attachments`, `sendMessage` cannot attach files; with it present, attachments flow through your adapter.

## Trade-offs to surface to the user

- **Funding model is now your concern.** With the HTTP adapter, the publisher pays the Walrus store tx and (typically) absorbs the cost. With this adapter, your `signer` pays in WAL tokens + SUI gas for every upload. Plan for: who funds the signer, refunds, and rate-limiting at your app layer.
- **Browser vs server.** The `@mysten/walrus` SDK works in both, but signing in the browser means the user pays per attachment (UX consideration) or you sponsor (extra infra).
- **Failover.** The HTTP adapter has a single point of failure (the publisher). With the SDK adapter you can configure multiple Walrus nodes via the `WalrusClient`, but you handle retries.
- **Latency / batching.** Quilts can batch multiple attachments into one store tx; preserve this by passing the whole `entries[]` to one Walrus store call rather than one tx per attachment.

## Verification

> **Safety:** any test that exercises `upload()` against a real Walrus network (testnet or mainnet) costs **real WAL + SUI** from the signer you wire in, and **mints persistent on-chain blob state**. Walrus storage cannot be "un-uploaded" — at best it expires after `epochs`. Run upload/download verification against (a) **mocked** Walrus client calls for fast unit tests, or (b) **testnet** with a dev signer funded from the faucet, never against mainnet during development. Never wire a production signer (or a multisig holding real value) into a verification harness.

1. **Unit test the adapter in isolation** with a **mocked `WalrusClient`** — assert that `upload()` calls the right SDK methods with the right arguments, that `ids[]` is returned in input order, and that `metadata` has the expected shape. No real network. Mirror [`ts-sdks/packages/sui-stack-messaging/test/unit/attachments-manager.test.ts`](../../../ts-sdks/packages/sui-stack-messaging/test/unit/attachments-manager.test.ts) for the test shape.
2. **Wire it into a localnet integration test** — copy [`ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts`](../../../ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts) and swap the storage adapter. If the local stack mocks Walrus, this is hermetic and free; if it points at testnet Walrus, a dev signer pays per upload. End-to-end pass means: encrypted bytes round-trip via your adapter and decrypt to the original plaintext.
3. **Confirm `metadata` shape parity** with `WalrusHttpStorageAdapter.WalrusUploadMetadata` if you want downstream code (e.g., deletion txs) to treat both adapters interchangeably. Non-mutating (you're comparing TypeScript shapes).
4. **One-time testnet smoke test** (optional, **mutating + costs WAL**): on a dev signer with testnet WAL + SUI, run one real upload + download of a small payload to confirm the end-to-end path works against actual Walrus. Treat this as a one-shot confidence check, not part of CI.

## Where this fits in the bigger picture

- The `StorageAdapter` interface and the canonical Walrus HTTP impl: `ts-sdks/packages/sui-stack-messaging/src/storage/`.
- Authoritative docs: [`docs/sui-stack-messaging/Attachments.md`](../../../docs/sui-stack-messaging/Attachments.md), [`docs/sui-stack-messaging/Extending.md`](../../../docs/sui-stack-messaging/Extending.md).
- Bigger Builder integration story: [`integrate-sui-stack-messaging`](../integrate-sui-stack-messaging/SKILL.md).

## Cross-links

- For *non*-Walrus backends (S3, IPFS): currently not a recommended path; the interface allows it but there's no canonical example. Open a GitHub issue if this is a hard requirement for your use case.
- `@mysten/walrus` SDK: https://www.npmjs.com/package/@mysten/walrus (use this skill in conjunction with its docs).
