# Developer Setup

## Contents

- [Architecture at a glance](#architecture-at-a-glance)
- [Prerequisites](#prerequisites)
- [Quick Setup with `createSuiStackMessagingClient`](#quick-setup-with-createsuistackmessagingclient)
- [Manual Extension Chain (Advanced)](#manual-extension-chain-advanced)
- [Configuration Reference](#configuration-reference)
- [Sub-Modules](#sub-modules)
- [The `GroupRef` Pattern](#the-groupref-pattern)
- [Where to go next](#where-to-go-next)

**Documentation:** [Home](../../README.md) | [Installation](./Installation.md) | [Examples](./Examples.md) | [API Reference](./APIRef.md) | [Encryption](./Encryption.md) | [Security](./Security.md) | [Relayer](./Relayer.md) | [Attachments](./Attachments.md) | [Archive & Recovery](./ArchiveRecovery.md) | [Group Discovery](./GroupDiscovery.md) | [Extending](./Extending.md) | [Testing](./Testing.md) | [Community Contributed Tools](./CommunityContributed.md)

---

## Architecture at a glance

The SDK is one client in a system that talks to four independent services. Encryption is end-to-end — none of them see plaintext.

| Service                                       | What it does                                                 | Who runs it                                                                                                                                                                       |
| --------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Sui RPC** (fullnode / gRPC)                 | Chain reads + transaction submission                         | Public endpoint or your own                                                                                                                                                       |
| **Relayer**                                   | Accepts ciphertext, indexes membership, serves messages back | **You run it.** No public canonical relayer exists — fork the reference at [`relayer/`](../../relayer/). See [Relayer](./Relayer.md).                                             |
| **Seal key servers**                          | Threshold-encrypt / decrypt DEKs                             | Mysten Labs operates the canonical allowlist; you reference their objectIds in `seal.serverConfigs`. See [Seal docs](https://github.com/MystenLabs/seal).                         |
| **Walrus publisher + aggregator** _(optional, attachments only)_ | Stores encrypted file bytes                                  | Public testnet endpoints, your own, or skip them entirely and talk to Walrus via the [`@mysten/walrus`](https://www.npmjs.com/package/@mysten/walrus) SDK (see [Extending](./Extending.md)). |

## Prerequisites

Before writing SDK code, confirm:

- [ ] **A running relayer** reachable from your app. For development, see [`relayer/README.md`](../../relayer/README.md); for production, fork the reference and operate your own.
- [ ] **Seal `serverConfigs`** — the canonical key-server object IDs for your network. See [Seal docs](https://github.com/MystenLabs/seal) for the current allowlist.
- [ ] **A Sui RPC URL** for the right network (testnet, mainnet, or localnet).
- [ ] **(Optional)** A Walrus publisher + aggregator if you need attachments, or commit to the `@mysten/walrus` SDK path (see [Attachments](./Attachments.md), [Extending](./Extending.md)).

---

This SDK follows the [MystenLabs TS SDK building guidelines](https://sdk.mystenlabs.com/sui/sdk-building). It uses the **client extension pattern**: you extend a base Sui client with messaging, groups, and Seal extensions.

## Quick Setup with `createSuiStackMessagingClient()`

This factory function composes all three extensions for you:

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { createSuiStackMessagingClient } from '@mysten/sui-stack-messaging';

const client = createSuiStackMessagingClient(
  new SuiGrpcClient({
    baseUrl: 'https://fullnode.testnet.sui.io:443',
    network: 'testnet',
  }),
  {
    seal: {
      serverConfigs: [
        { objectId: '0x...', weight: 1 },
        { objectId: '0x...', weight: 1 },
      ],
    },
    encryption: {
      sessionKey: { signer: keypair },
    },
    relayer: {
      relayerUrl: 'https://your-relayer.example.com',
    },
  },
);
```

After creation, the client exposes four namespaces:

| Namespace          | Purpose                                                                             |
| ------------------ | ----------------------------------------------------------------------------------- |
| `client.messaging` | E2EE messaging, group creation, key rotation                                        |
| `client.groups`    | Permission management ([Sui Groups docs](https://github.com/MystenLabs/sui-groups)) |
| `client.seal`      | Seal encryption / decryption (used by `messaging`)                                  |
| `client.core`      | Base Sui RPC methods                                                                |

## Manual Extension Chain (Advanced)

For custom extension names or full control, use `$extend()` directly:

```typescript
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { SealClient } from '@mysten/seal';
import { suiGroups } from '@mysten/sui-groups';
import { suiStackMessaging } from '@mysten/sui-stack-messaging';

const base = new SuiGrpcClient({
  baseUrl: 'https://fullnode.testnet.sui.io:443',
  network: 'testnet',
});

// Step 1: extend with groups + seal (independent of each other)
const withGroupsAndSeal = base.$extend(
  suiGroups({
    witnessType: `${MESSAGING_PACKAGE_ID}::messaging::Messaging`,
  }),
  {
    name: 'seal' as const,
    register: (c) =>
      new SealClient({
        suiClient: c,
        serverConfigs: [
          { objectId: '0x...', weight: 1 },
          { objectId: '0x...', weight: 1 },
        ],
      }),
  },
);

// Step 2: extend with messaging (depends on both groups and seal)
const client = withGroupsAndSeal.$extend(
  suiStackMessaging({
    encryption: { sessionKey: { signer: keypair } },
    relayer: { relayerUrl: 'https://your-relayer.example.com' },
  }),
);
```

## Configuration Reference

### `encryption` (required)

Controls how the SDK obtains Seal session keys and encrypts / decrypts messages.

#### Session Key Tiers

**Tier 1: Signer-based** (recommended; works with `@mysten/dapp-kit`'s `CurrentAccountSigner`, `Keypair`, Enoki):

```typescript
encryption: {
  sessionKey: { signer: keypair },
}
```

The SDK derives the address via `signer.toSuiAddress()`, creates a `SessionKey`, and handles certification automatically.

**Tier 2: Callback-based** (when you only have a `signPersonalMessage` surface, not a full `Signer`):

```typescript
encryption: {
  sessionKey: {
    address: '0x...',
    onSign: async (message: Uint8Array) => {
      // Sign with your wallet adapter and return the signature string
      return signPersonalMessage(message);
    },
  },
}
```

The SDK creates the session key, then calls `onSign()` with the personal-message bytes.

**Tier 3: Consumer-managed** (full control over the `SessionKey` lifecycle):

```typescript
encryption: {
  sessionKey: {
    getSessionKey: () => myManagedSessionKey,
  },
}
```

#### Session Key Options (Tier 1 & 2)

| Option            | Default  | Description                                              |
| ----------------- | -------- | -------------------------------------------------------- |
| `ttlMin`          | `10`     | Session-key TTL in minutes; also sets the DEK cache TTL  |
| `refreshBufferMs` | `60_000` | Refresh this many ms before expiry                       |
| `mvrName`         | (none)   | MVR name for Seal package resolution                     |

Tier 3's variant excludes these options; its DEK cache falls back to the 10-minute default regardless of your `SessionKey`'s own TTL.

#### Encryption Options

| Option             | Default             | Description                                                         |
| ------------------ | ------------------- | ------------------------------------------------------------------- |
| `sealThreshold`    | `2`                 | Number of key servers needed for decryption                         |
| `cryptoPrimitives` | Web Crypto          | Custom AES-GCM implementation                                       |
| `sealPolicy`       | `DefaultSealPolicy` | Custom Seal access-control policy (see [Extending](./Extending.md)) |

### `relayer` (required)

**The relayer must be running and reachable before any message operation succeeds.** `sendMessage`, `getMessages`, `subscribe`, and friends all go through it. No public canonical relayer exists — see [`relayer/README.md`](../../relayer/README.md) to spin up the reference for development, or fork it for production.

Either provide a URL for the built-in HTTP transport, or supply a custom transport instance:

```typescript
// Built-in HTTP transport
relayer: {
  relayerUrl: 'https://your-relayer.example.com',
  pollingIntervalMs: 3000,  // default
  timeout: 30_000,          // default (ms)
  onError: (err) => console.error(err),
}

// Custom transport
relayer: {
  transport: myCustomTransport,  // implements RelayerTransport
}
```

See [Relayer](./Relayer.md) for the wire protocol and `RelayerTransport` interface.

### `attachments` (optional)

Enable file-attachment support by providing a storage adapter:

```typescript
import { WalrusHttpStorageAdapter } from '@mysten/sui-stack-messaging';

attachments: {
  storageAdapter: new WalrusHttpStorageAdapter({
    publisherUrl: 'https://publisher.walrus-testnet.walrus.space',
    aggregatorUrl: 'https://aggregator.walrus-testnet.walrus.space',
    epochs: 5,
  }),
  maxAttachments: 10,                // default
  maxFileSizeBytes: 10_485_760,      // 10 MB default
  maxTotalFileSizeBytes: 52_428_800, // 50 MB default
}
```

When omitted, `sendMessage` cannot include files and received attachment metadata is not resolvable. See [Attachments](./Attachments.md) — and [Extending](./Extending.md) for non-HTTP / `@mysten/walrus`-SDK adapters.

### `packageConfig` (optional)

Auto-detected for testnet and mainnet. Required for localnet or custom deployments. **The shape differs between the factory and the manual chain.**

**Factory** (`createSuiStackMessagingClient`):

```typescript
packageConfig: {
  messaging: {
    originalPackageId: '0x...',  // First published — type names, BCS, Seal namespace
    latestPackageId: '0x...',    // Current — moveCall targets
    namespaceId: '0x...',        // MessagingNamespace shared object
    versionId: '0x...',          // Version shared object
  },
  permissionedGroups: {          // optional, also auto-detected
    originalPackageId: '0x...',
    latestPackageId: '0x...',
  },
}
```

**Manual** (`suiStackMessaging({...})`): a flat `SuiStackMessagingPackageConfig` — just the four messaging fields above. The `permissionedGroups` config is passed separately into `suiGroups({ packageConfig })` within the same `$extend` call.

### `suinsConfig` (optional)

Auto-detected for testnet and mainnet. Only needed for SuiNS reverse-lookup operations (`setSuinsReverseLookup`, `unsetSuinsReverseLookup`).

### `seal` (factory only)

When using `createSuiStackMessagingClient`, pass either a pre-built `SealClient` or Seal config options:

```typescript
// Config options (SealClient created internally)
seal: {
  serverConfigs: [
    { objectId: '0x...', weight: 1 },
    { objectId: '0x...', weight: 1 },
  ],
}

// Pre-built SealClient
seal: existingSealClient,
```

In the manual chain you construct the `SealClient` directly inside the `$extend` `register` callback (see the example above).

## Sub-Modules

The `client.messaging` object exposes several sub-modules:

| Sub-module   | Purpose                                  | Example                                                   |
| ------------ | ---------------------------------------- | --------------------------------------------------------- |
| `call`       | PTB thunks: composable transaction steps | `tx.add(client.messaging.call.createAndShareGroup(opts))` |
| `tx`         | Full transactions: ready to sign         | `client.messaging.tx.createAndShareGroup(opts)`           |
| `view`       | Read-only queries (no gas)               | `client.messaging.view.groupsMetadata(opts)`              |
| `bcs`        | BCS type definitions for parsing         | `client.messaging.bcs.EncryptionHistory`                  |
| `derive`     | Deterministic address derivation         | `client.messaging.derive.groupId({ uuid })`               |
| `encryption` | Low-level encrypt / decrypt              | `client.messaging.encryption.encrypt(opts)`               |
| `transport`  | Direct relayer access                    | `client.messaging.transport.fetchMessages(opts)`          |

### When to use which

- **Top-level imperative methods** (e.g. `client.messaging.sendMessage()`) — simplest path: sign, encrypt, and send in one call.
- **`tx.*`** — when you need a `Transaction` object to inspect or modify before signing (e.g. with dapp-kit's `signAndExecuteTransaction`).
- **`call.*`** — when composing multiple operations into a single PTB.
- **`view.*`** — for read-only queries that don't require a signer.

## The `GroupRef` Pattern

Most messaging methods accept a `GroupRef`, either a UUID or explicit object IDs:

```typescript
// By UUID (simpler: derives both IDs internally)
await client.messaging.sendMessage({
  signer: keypair,
  groupRef: { uuid: 'my-group-uuid' },
  text: 'Hello!',
});

// By explicit IDs
await client.messaging.sendMessage({
  signer: keypair,
  groupRef: {
    groupId: '0x...',
    encryptionHistoryId: '0x...',
  },
  text: 'Hello!',
});
```

Using UUIDs is recommended. See [Group Discovery](./GroupDiscovery.md) for details on UUID derivation and tracking.

## Where to go next

- [Examples](./Examples.md) — end-to-end usage patterns.
- [Relayer](./Relayer.md) — wire protocol and custom transports.
- [Encryption](./Encryption.md) / [Security](./Security.md) — trust model.
- [Attachments](./Attachments.md) / [Extending](./Extending.md) — custom storage, custom Seal policies, custom recovery.
- [Testing](./Testing.md) — localnet docker stack and integration tests.
