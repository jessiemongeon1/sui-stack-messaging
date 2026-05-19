---
name: configure-walrus-storage-via-sdk
description: Use when a developer wants to implement a custom attachments `StorageAdapter` for sui-stack-messaging — most commonly talking to Walrus directly via the `@mysten/walrus` SDK (instead of going through a publisher / aggregator), or plugging in a non-Walrus backend like S3. Covers the scope (attachments only, not message archive/recovery), why you'd pick the SDK path (no runtime dep on publisher/aggregator infra, fine-grained control over Walrus flows, support for Upload Relay, access to capabilities publishers don't expose like deletion and epoch-extension), and how to wire your adapter into the messaging client. Trigger phrases - "custom Walrus storage", "use @mysten/walrus directly", "programmatic Walrus", "Walrus SDK adapter", "implement StorageAdapter", "upload relay", "delete blobs from messaging", "extend blob epochs", "S3 attachments", "non-Walrus storage", "skip publisher".
---

# Custom attachments storage via the `@mysten/walrus` SDK

The canonical attachments path uses `WalrusHttpStorageAdapter` (`ts-sdks/packages/sui-stack-messaging/src/storage/walrus-http-storage-adapter.ts`), which talks to a Walrus **publisher** (HTTP `PUT /v1/quilts`) and **aggregator** (HTTP `GET /v1/blobs/by-quilt-patch-id/{id}`). That works, but a runtime dependency on a publisher/aggregator service is something you may not want — either because you'd rather not run/pay for that infra at all, or because you want capabilities the publisher's HTTP API doesn't expose.

This skill covers two alternatives:

1. **Talk to Walrus through the `@mysten/walrus` TS SDK** — sign and submit Walrus register/certify/delete/extend transactions yourself (your app code plays the publisher role). Optionally use a Walrus **Upload Relay** to offload the bulk-write fan-out while you keep signing the txs.
2. **Plug in a non-Walrus backend** (S3, IPFS, etc.) — the `StorageAdapter` interface is genuinely backend-agnostic.

## Scope of this skill — attachments only

The `StorageAdapter` interface is consumed by `AttachmentsManager` and handles **only attachments**: the encrypted bytes that ride alongside messages (images, files, etc.). It is decoupled by design from how *messages themselves* are persisted — message bytes flow through the relayer wire protocol, and long-term message recoverability is a separate concern handled by the relayer's Walrus archive (see `docs/sui-stack-messaging/ArchiveRecovery.md`).

So: if you want to customize attachment storage, you only need to implement `StorageAdapter` on the SDK side and hand it to the messaging client. **No relayer changes required.** Message archival and recovery are an orthogonal axis.

## Walrus is the recommendation, not a requirement

Walrus is the storage layer we recommend and build around — the canonical adapter targets it, the docs assume it, and the on-chain attachment metadata (`Attachment` Move type and friends) is shaped to round-trip Walrus blob IDs cleanly.

That said, the `StorageAdapter` interface deliberately uses `unknown` for adapter-specific metadata and decouples encryption from storage, so a non-Walrus backend (S3, an internal object store, etc.) plugs in just as well. The JSDoc on `storage-adapter.ts` calls this out explicitly. If you have a strong reason to use a non-Walrus backend, this skill's "wire it into the messaging client" section applies unchanged — only the body of `upload`/`download`/`delete` changes.

## Why pick the SDK path over a publisher/aggregator

A note on framing: "not the public Walrus publishers/aggregators" is not the same as "use the SDK." You can also self-host a publisher/aggregator, or pay for a managed one — in which case all the publisher-shaped limitations below still apply. The SDK path is its own choice, with its own value proposition:

- **No runtime dependency on publisher/aggregator infra at all.** Your app speaks to Walrus storage nodes (or an Upload Relay) directly; you don't have one more service to operate, monitor, scale, or pay a third party for. The `WalrusClient` you configure replaces both the publisher and the aggregator from your stack diagram.
- **Fine-grained control over Walrus flows and transactions.** The SDK exposes the full register → upload → certify pipeline (`writeFilesFlow` / `writeBlobFlow`), separable user-gesture steps for browser wallets that block popups, crash-recoverable resumable uploads (`onStep` + `resume`), custom `fetch` for timeouts/retries/dispatchers, error-class introspection (`RetryableWalrusClientError`), and the ability to issue your own register/certify/delete/extend transactions in whatever PTB shape your app needs.
- **Access to capabilities the publisher HTTP API doesn't expose.** From the publisher OpenAPI (`PUT /v1/blobs`, `PUT /v1/quilts`): no `DELETE`, no `EXTEND`, no register/certify split, no `onStep`/resume. If you need to delete `deletable` blobs (e.g., on attachment retraction), extend the storage epochs of an existing blob, or persist intermediate write state for crash recovery, the SDK gives you those; the publisher does not.
- **Upload Relay support.** The Walrus Upload Relay (configured via `walrus({ uploadRelay: { host, sendTip } })`) sits *between* you and the storage nodes: it absorbs the bulk-write fan-out (~2200 storage-node requests per blob) and turns it into a single request your app makes. Crucially — and this is the difference from a publisher — **your signer still signs and pays for the register and certify Sui transactions.** The relay just does the storage-node IO and takes an optional MIST tip for the work. Reads still go directly to storage nodes; the relay does not act as an aggregator.

## Tradeoffs — be aware, then move on

- Direct storage-node reads/writes are request-heavy (~2200 to write, ~335 to read per blob without an Upload Relay); plan for latency and concurrency at your app layer.
- Your signer pays per upload in WAL + SUI. With a publisher the publisher typically absorbed that — without it, you fund the signer, sponsor the txs, or charge the user.
- `@mysten/walrus` pulls in a WASM module for encoding/decoding. Some bundlers (Vite, Next.js) need explicit configuration — see the SDK README's wasm-loading section.

These matter, but they're well-trodden ground in the `@mysten/walrus` README. This skill is about *how* to wrap the SDK in a `StorageAdapter`, not how to operate Walrus.

## Where this runs

"Client-side" can be misleading here — the messaging TS SDK runs in browser and server runtimes equally well. Pick by **who signs**:

- **Browser-side signer** (the end user's wallet): use `writeFilesFlow` / `writeBlobFlow` so register and certify happen in separate user gestures and the wallet popup isn't blocked. User pays per attachment (UX implication).
- **Server-side signer** (a key your backend custodies, or a sponsor): use the higher-level `writeFiles` / `writeBlob`; you control retries, batching, and funding. Your app pays per attachment; you charge or absorb the cost.

Both are first-class. The adapter implementation barely changes.

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

Construct the Walrus-extended Sui client in your app bootstrap and hand it to the adapter. This is the same client-extension pattern the SDK uses for `seal()` and `suiGroups()` — keep it consistent on the Builder side.

```ts
import type {
  StorageAdapter,
  StorageEntry,
  StorageUploadResult,
} from '@mysten/sui-stack-messaging';
import { WalrusFile, walrus } from '@mysten/walrus';
import type { Signer } from '@mysten/sui/cryptography';
import { SuiGrpcClient } from '@mysten/sui/grpc';

// In your app bootstrap, once:
const walrusExtendedClient = new SuiGrpcClient({
  network: 'testnet',
  baseUrl: 'https://fullnode.testnet.sui.io:443',
}).$extend(walrus(/* optional { uploadRelay: { host, sendTip } } */));

export interface WalrusSdkAdapterConfig {
  walrusClient: typeof walrusExtendedClient;  // already walrus()-extended
  signer: Signer;       // signs register + certify + (optionally) delete/extend txs
  epochs: number;       // storage duration ahead of current epoch
  deletable?: boolean;  // if true, the resulting Blob objects support `delete`
}

export interface WalrusSdkUploadMetadata {
  // Mirror WalrusUploadMetadata shape from WalrusHttpStorageAdapter so downstream
  // code (deletion, epoch extension) can treat both adapters interchangeably.
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
    // Bundle entries into one quilt — same shape as the canonical HTTP adapter,
    // one Sui Blob object covering all attachments in this batch.
    const files = entries.map((e) =>
      WalrusFile.from({ contents: e.data, identifier: e.name }),
    );

    // Server-side / full-control path: writeFiles. For browser wallets that need
    // separate user gestures, swap in walrusClient.walrus.writeFilesFlow and
    // call register/upload/certify from distinct event handlers.
    const results = await this.cfg.walrusClient.walrus.writeFiles({
      files,
      epochs: this.cfg.epochs,
      deletable: this.cfg.deletable ?? true,
      signer: this.cfg.signer,
    });

    return {
      ids: results.map((r) => r.id),           // quilt patch IDs, positionally aligned
      metadata: { /* WalrusSdkUploadMetadata derived from results[0].blobObject */ },
    };
  }

  async download(id: string): Promise<Uint8Array> {
    const [file] = await this.cfg.walrusClient.walrus.getFiles({ ids: [id] });
    return file.bytes();
  }

  async delete(ids: string[]): Promise<void> {
    // Only meaningful if your blobs were created with `deletable: true`.
    // Build + sign + submit the Walrus `delete` transactions for the corresponding
    // Sui Blob objects. Group by quilt blobObjectId from your persisted metadata
    // — one Sui Blob covers many quilt patches.
  }
}
```

Notes on the underlying SDK choices (consult the [`@mysten/walrus` README](https://www.npmjs.com/package/@mysten/walrus) for full APIs):

- **`writeFiles` vs `writeFilesFlow`.** `writeFiles` is one-shot, ideal for server-side signers. `writeFilesFlow` returns `encode` / `register` / `upload` / `certify` / `listFiles` separately — required when the signer is a browser wallet that pops up for each tx.
- **`writeBlob` / `writeBlobFlow`.** Lower-level if you don't want quilts (one blob per attachment); supports `onStep` + `resume` for crash-recovery. The canonical HTTP adapter quilts, so quilting keeps adapter parity.
- **Upload Relay.** Configured at `walrus(...)` time, not per call: `walrus({ uploadRelay: { host, sendTip: { max: 1_000 } } })`. The relay handles fan-out to storage nodes; your signer still signs register + certify and pays storage fees in WAL. The tip is paid in MIST.

## Wire it into the messaging client

Pass an instance under `attachments.storageAdapter`:

```ts
const client = createSuiStackMessagingClient(baseClient, {
  // ...seal, encryption, relayer
  attachments: {
    storageAdapter: new WalrusSdkStorageAdapter({
      walrusClient: walrusExtendedClient,
      signer: attachmentSigner,
      epochs: 5,
      deletable: true,
    }),
  },
});
```

`WalrusSdkStorageAdapter` and `WalrusSdkUploadMetadata` above are *your* implementation's names — choose what fits. The canonical reference to mirror for shape parity is `WalrusHttpStorageAdapter` / `WalrusUploadMetadata` in [`storage/walrus-http-storage-adapter.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/storage/walrus-http-storage-adapter.ts).

Without `attachments`, `sendMessage` cannot attach files; with it present, attachments flow through your adapter. Message text itself is unaffected — it does not pass through `StorageAdapter` and continues to flow through the relayer wire protocol regardless.

## Non-Walrus backends (S3, etc.) in one paragraph

The same skeleton works: replace the `WalrusClient` and `Signer` with whatever SDK your backend uses, return your backend's identifiers as `ids[]`, and define a `metadata` shape that lets you find the bytes again later. The `StorageAdapter` JSDoc explicitly mentions S3 as a legitimate target. The only Walrus-specific concept that doesn't carry over is the on-chain blob lifecycle (epoch extension, deletion via Move call) — for an S3-style backend you'd model deletion through your backend's own APIs and probably leave the SDK's adapter-level metadata empty.

## Verification

> **Safety:** any test that exercises `upload()` against a real Walrus network (testnet or mainnet) costs **real WAL + SUI** from the signer you wire in, and **mints persistent on-chain blob state**. Walrus storage cannot be "un-uploaded" — at best it expires after `epochs`, or you call `delete` if the blob was created `deletable: true`. Run upload/download verification against (a) **mocked** Walrus client calls for fast unit tests, or (b) **testnet** with a dev signer funded from the faucet, never against mainnet during development. Never wire a production signer (or a multisig holding real value) into a verification harness.

1. **Unit test the adapter in isolation** with a **mocked Walrus client** — assert `upload()` calls `writeFiles` (or `writeFilesFlow` for browser flow) with the right arguments, that `ids[]` is returned in input order, and that `metadata` has the expected shape. No real network. Mirror [`ts-sdks/packages/sui-stack-messaging/test/unit/attachments-manager.test.ts`](../../../ts-sdks/packages/sui-stack-messaging/test/unit/attachments-manager.test.ts) for the test shape.
2. **Localnet integration test** — copy [`ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts`](../../../ts-sdks/packages/sui-stack-messaging/test/integration/localnet/flows.test.ts) and swap the storage adapter. If the local stack mocks Walrus, this is hermetic and free; if it points at testnet Walrus, a dev signer pays per upload. End-to-end pass: encrypted bytes round-trip via your adapter and decrypt to the original plaintext.
3. **Confirm `metadata` shape parity** with the canonical adapter if you want downstream code (deletion txs, epoch extension) to treat both adapters interchangeably. Non-mutating (you're comparing TypeScript shapes).
4. **One-time testnet smoke test** (optional, **mutating + costs WAL**): on a dev signer with testnet WAL + SUI, run one real upload + download of a small payload against actual Walrus storage nodes (and, if you've enabled it, against an Upload Relay). Treat this as a one-shot confidence check, not part of CI.

## Cross-links

- **`StorageAdapter` interface + canonical HTTP impl:** `ts-sdks/packages/sui-stack-messaging/src/storage/`.
- **Authoritative attachment docs:** [`docs/sui-stack-messaging/Attachments.md`](../../../docs/sui-stack-messaging/Attachments.md), [`docs/sui-stack-messaging/Extending.md`](../../../docs/sui-stack-messaging/Extending.md).
- **Message archival (separate concern):** [`docs/sui-stack-messaging/ArchiveRecovery.md`](../../../docs/sui-stack-messaging/ArchiveRecovery.md).
- **`@mysten/walrus` SDK:** https://www.npmjs.com/package/@mysten/walrus — full API, Upload Relay configuration, browser wasm setup, error classes.
- **Walrus publisher API surface (for reference, what the SDK path avoids depending on):** publisher OpenAPI exposes only `PUT /v1/blobs` and `PUT /v1/quilts` — no delete, no extend.
- **Bigger Builder integration story:** [`integrate-sui-stack-messaging`](../integrate-sui-stack-messaging/SKILL.md).
