---
name: integrate-sui-stack-messaging
description: Use when a developer wants to add Sui Stack Messaging to their own dapp / project — installing the @mysten/sui-stack-messaging npm package, picking the createSuiStackMessagingClient factory vs manual $extend chain, wiring up session keys, and the minimum config to send one message. Trigger phrases - "add messaging to my app", "install sui-stack-messaging", "integrate the SDK", "how do I use sui-stack-messaging from React/Next/Node", "minimum setup to send a message", "@mysten/sui-stack-messaging quickstart", "wire up the messaging client".
---

# Integrate Sui Stack Messaging into your dapp

You are consuming the canonical npm SDK in your own application. You are **not** modifying anything in this repo. (If you want to fork the relayer or indexer, use [`develop-relayer`](../develop-relayer/SKILL.md) / [`develop-walrus-indexer`](../develop-walrus-indexer/SKILL.md) instead.)

## Hard prerequisite — flag this first

**Message-related operations require a running relayer.** `sendMessage`, attachment upload, fetching messages, etc. all go through the relayer's HTTP wire protocol. Without one configured and reachable, those calls will fail.

Two ways to satisfy this prerequisite:

1. **For local dev:** run the reference Rust relayer locally → see [`spin-up-relayer`](../spin-up-relayer/SKILL.md). Point your client at `http://localhost:3000`.
2. **For production:** point at a deployed relayer (your own deployment of the reference, a forked Rust relayer, or a custom `RelayerTransport` you implemented) → see [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md).

There is no zero-server path. Even the simplest example below assumes a relayer URL is reachable.

## Install

```bash
pnpm add @mysten/sui-stack-messaging @mysten/sui-groups
# Peer deps (most Sui dapps already have these):
pnpm add @mysten/seal @mysten/sui @mysten/bcs
```

Peer dep minimums (verify against [`ts-sdks/packages/sui-stack-messaging/package.json`](../../../ts-sdks/packages/sui-stack-messaging/package.json) — source is truth, `Installation.md` may lag): `@mysten/seal ^1.1.1`, `@mysten/sui ^2.6.0`, `@mysten/bcs ^2.0.2`. Node ≥ 22, pnpm ≥ 10.17.0.

## Pick a setup path

Two equivalent paths. Default to the factory unless you specifically need per-extension control.

### Path A (recommended) — `createSuiStackMessagingClient` factory

One call handles `suiGroups` + `seal` + `suiStackMessaging` extension chaining for you. The exact exported name is `createSuiStackMessagingClient` ([`ts-sdks/packages/sui-stack-messaging/src/factory.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/factory.ts)). Some docs may show an older name (`createMessagingGroupsClient`) — that's stale; trust the source.

```ts
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
    encryption: { sessionKey: { signer: keypair } },
    relayer: { relayerUrl: 'https://your-relayer.example.com' },
  },
);
```

After construction, four namespaces are available: `client.messaging` (E2EE messaging), `client.groups` (permissions, [Sui Groups](https://github.com/MystenLabs/sui-groups)), `client.seal` (encryption), `client.core` (base Sui RPC).

### Path B (advanced) — manual `$extend` chain

Use this only if you need separate references to the intermediate clients, or if you're injecting a pre-built `SealClient`. The chain order matters: `suiGroups` + `seal` first, then `suiStackMessaging` on top. The canonical shape (mirrored from [`factory.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/factory.ts)) — note `seal` is registered inline as `{ name: 'seal' as const, register: (c) => new SealClient({...}) }`, not via a `seal({...})` factory:

```ts
import { SuiGrpcClient } from '@mysten/sui/grpc';
import { SealClient } from '@mysten/seal';
import { suiGroups } from '@mysten/sui-groups';
import { suiStackMessaging } from '@mysten/sui-stack-messaging';

const client = new SuiGrpcClient({
  baseUrl: 'https://fullnode.testnet.sui.io:443',
  network: 'testnet',
})
  .$extend(
    suiGroups({ witnessType: `${MESSAGING_PACKAGE_ID}::messaging::Messaging` }),
    {
      name: 'seal' as const,
      register: (c) => new SealClient({ suiClient: c, serverConfigs: [...] }),
    },
  )
  .$extend(
    suiStackMessaging({
      encryption: { sessionKey: { signer: keypair } },
      relayer: { relayerUrl: 'https://your-relayer.example.com' },
    }),
  );
```

See also [`docs/sui-stack-messaging/Setup.md`](../../../docs/sui-stack-messaging/Setup.md), but cross-check any factory/extension names against [`src/factory.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/factory.ts) and [`src/client.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/client.ts) — that doc is known to lag the source.

## Picking session-key tier

Three tiers, pick by your auth situation:

| Tier | When | What you pass |
|---|---|---|
| **1 — signer-based** | dapp-kit-next, Keypair, Enoki, server-side Node with a key | `encryption: { sessionKey: { signer: keypair } }` |
| **2 — callback-based** | current dapp-kit without the Signer abstraction | `encryption: { sessionKey: { address, onSign: async (msg) => signPersonalMessage(msg) } }` |
| **3 — manual** | you already manage `SessionKey` lifecycle externally | `encryption: { sessionKey: { getSessionKey: () => myManaged } }` |

Tier-1 is the default. Tier-2 is what most current React-wallet integrations end up needing. Tier-3 is rare — only choose if you have a strong reason to manage the key elsewhere.

For deeper coverage (TTL, refresh buffer, key rotation, MVR), see [`configure-session-keys`](../configure-session-keys/SKILL.md).

## Network auto-detection

Package IDs for `sui_stack_messaging` are auto-detected for testnet and mainnet — you don't need to pass `packageConfig` unless you're on localnet or a custom deployment. Constants live in [`ts-sdks/packages/sui-stack-messaging/src/constants.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/constants.ts).

For localnet, pass an explicit `packageConfig` — see the `Configuration Reference` section of [`Setup.md`](../../../docs/sui-stack-messaging/Setup.md).

## Minimal "send + receive one message" example

```ts
// 1. Create a group (one-time setup)
const { groupId, encryptionHistoryId } = await client.messaging.createAndShareGroup({
  signer: keypair,
  groupRef: { uuid: 'my-app-channel-1' },
});

// 2. Send
await client.messaging.sendMessage({
  signer: keypair,
  groupRef: { uuid: 'my-app-channel-1' },
  text: 'Hello!',
});

// 3. Fetch
const { messages } = await client.messaging.getMessages({
  groupRef: { uuid: 'my-app-channel-1' },
  limit: 20,
});
```

The `GroupRef` pattern (UUID-derived IDs vs explicit object IDs) is documented in `Setup.md#groupref-pattern`. UUIDs are the recommended default.

## Optional: attachments

By default `sendMessage` cannot attach files. To enable file attachments, configure a storage adapter:

```ts
import { WalrusHttpStorageAdapter } from '@mysten/sui-stack-messaging';

createSuiStackMessagingClient(base, {
  // ...other config
  attachments: {
    storageAdapter: new WalrusHttpStorageAdapter({
      publisherUrl: 'https://publisher.walrus-testnet.walrus.space',
      aggregatorUrl: 'https://aggregator.walrus-testnet.walrus.space',
      epochs: 5,
    }),
  },
});
```

If you don't want to depend on public Walrus publishers/aggregators, see [`configure-walrus-storage-via-sdk`](../configure-walrus-storage-via-sdk/SKILL.md) for the programmatic alternative.

## Verification — the order to test things in

> **Safety:** steps (3) and (4) are **mutating** — they create a real on-chain group and write a real message via the relayer. Run them with a **dev signer on testnet (or localnet)** against a **throwaway group** (e.g. `uuid: \`integration-test-${Date.now()}\``). Never verify against a production group, a production relayer, or a wallet holding real value. None of the on-chain side effects can be undone.

1. **Client constructs without throwing.** Sanity check imports, peer deps, and `serverConfigs`. Non-mutating.
2. **Relayer reachable.** Independently: `curl http://localhost:3000/health_check` (dev) or your prod equivalent. Non-mutating.
3. **`createAndShareGroup` returns IDs.** Confirms signer + Sui RPC work. **Mutating — creates a shared on-chain group object; use a fresh dev UUID.**
4. **`sendMessage` returns without throwing.** Confirms relayer auth + encryption. **Mutating — writes ciphertext via the relayer (and, depending on relayer config, may touch on-chain state too).**
5. **`getMessages` returns the message you just sent, with `text` correctly decrypted.** Confirms full round-trip including Seal session-key flow. Non-mutating (read-only).

If any step past (1) fails, the most common causes are:
- (2) failure → relayer not running, wrong URL, CORS misconfig.
- (3) failure → signer doesn't have gas; wrong network in `SuiGrpcClient`; localnet without `packageConfig`.
- (4) failure → relayer's `GROUPS_PACKAGE_ID` doesn't match the network you're on; Seal `serverConfigs` invalid; session key creation failed.
- (5) decrypt failure → see [`debug-encryption-flow`](../debug-encryption-flow/SKILL.md).

## Where to go next

- Custom Seal access control (token-gating, subscriptions): [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md).
- Bring your own relayer instead of running the reference Rust one: [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md).
- Programmatic Walrus storage (skip public publishers/aggregators): [`configure-walrus-storage-via-sdk`](../configure-walrus-storage-via-sdk/SKILL.md).
- Production-ready session key handling: [`configure-session-keys`](../configure-session-keys/SKILL.md).
- Debug decryption / encryption failures: [`debug-encryption-flow`](../debug-encryption-flow/SKILL.md).

## Authoritative docs

- [`docs/sui-stack-messaging/Installation.md`](../../../docs/sui-stack-messaging/Installation.md) — install + peer deps.
- [`docs/sui-stack-messaging/Setup.md`](../../../docs/sui-stack-messaging/Setup.md) — full configuration reference.
- [`docs/sui-stack-messaging/Examples.md`](../../../docs/sui-stack-messaging/Examples.md) — end-to-end usage flows.
- [`docs/sui-stack-messaging/APIRef.md`](../../../docs/sui-stack-messaging/APIRef.md) — every public method.
