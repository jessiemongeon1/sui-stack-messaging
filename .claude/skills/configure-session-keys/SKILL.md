---
name: configure-session-keys
description: Use when a developer wants to pick or configure the Seal session-key strategy for their Sui Stack Messaging integration — the three tiers (signer / callback / manual), TTL and refresh-buffer tuning, lifecycle management, key rotation under load, MVR-name resolution. Critical decision for any Builder integrating wallet auth. Trigger phrases - "session key management", "sessionKeyConfig", "Seal session key", "signer vs onSign", "long-lived session", "session key TTL", "session key refresh", "wallet session for messaging", "personal message signing for messaging".
---

# Configure Seal session keys

Every Seal decryption call requires a short-lived, certified **session key** that authorizes the SDK to ask the key servers to release the DEK. The session key is one-per-(user, package) and lives only in memory. The SDK abstracts the lifecycle through three tiers — picking the right one and tuning TTL is one of the highest-impact decisions in any Builder integration.

## The three tiers — pick by your auth surface

```ts
// All three live under `encryption: { sessionKey: ... }` in the messaging client config.
```

### Tier 1 — signer-based (recommended default)

```ts
encryption: { sessionKey: { signer: keypair } }
```

The SDK derives the address from `signer.toSuiAddress()`, creates a `SessionKey`, and certifies it automatically. Works with:

- `Keypair` directly (Node/server-side, scripts).
- `@mysten/dapp-kit`'s `CurrentAccountSigner`.
- Enoki's `EnokiSigner`.

**Use this whenever you have a `Signer` instance.** Zero ceremony.

### Tier 2 — callback-based

```ts
encryption: {
  sessionKey: {
    address: '0x...',
    onSign: async (message: Uint8Array) => {
      // Your wallet adapter signs the personal message and returns the signature string.
      return signPersonalMessage(message);
    },
  },
}
```

The SDK creates the session key, then calls `onSign()` with the personal-message bytes. Use this when you only have a "sign personal message" surface — not a full `Signer` — to integrate with.

UX note: `onSign` triggers a wallet popup. The first decryption per session will prompt the user. Tune `ttlMin` (below) to avoid re-prompting.

### Tier 3 — consumer-managed SessionKey

```ts
encryption: { sessionKey: { getSessionKey: () => myManagedSessionKey } }
```

You own creation, certification, expiry, and replacement of the `SessionKey`. The callback may return `SessionKey` or `Promise<SessionKey>`. The manager caches the returned instance and re-calls your callback only when it detects the cached instance is approaching its own `ttlMin` expiry. Useful for sharing one SessionKey across multiple SDK instances, persisting/rehydrating across page reloads, or wiring in custom creation logic.

Note: Tier 3's config variant excludes `ttlMin`, so the DEK cache falls back to the 10-minute default regardless of your SessionKey's own TTL — a 30-min consumer-managed session will still see DEK entries flush every 10 min.

## TTL and refresh-buffer tuning (Tier 1 & 2)

| Option | Default | Description | When to change |
|---|---|---|---|
| `ttlMin` | `10` (minutes) | Session-key lifetime; **also** the DEK cache TTL (same value backs both for Tier 1/2) | Increase to reduce wallet prompts and Seal round-trips; decrease for tighter access-revocation latency |
| `refreshBufferMs` | `60000` (60 sec) | Refresh proactively this many ms before expiry | Increase if you've seen "key expired during long fetches"; decrease to push TTL boundary |
| `mvrName` | `undefined` | MVR (Move Registry) name for Seal policy resolution | Set only if you've registered your package under MVR; otherwise leave undefined |

Trade-off: longer `ttlMin` = fewer wallet popups, but a longer window where a stolen session key can decrypt. The default of 10 minutes is a reasonable starting point for most consumer dapps. For high-frequency / power-user flows where popups are painful, 30–60 minutes is reasonable.

Note: the session key only authorizes **decryption requests to Seal key servers** — it does not authorize on-chain mutations. Even with a long TTL, every state-changing tx still requires the underlying signer / wallet.

## Lifecycle: what happens behind the scenes

Whichever tier you pick, the SDK's `SessionKeyManager` (`ts-sdks/packages/sui-stack-messaging/src/encryption/session-key-manager.ts`) handles:

1. **Lazy creation.** No session key is created until the first encryption / decryption call needs one. Avoids prompting the wallet on idle pages.
2. **Single-flight refresh.** When the current key is within `refreshBufferMs` of expiry, the next call triggers creation of a new one. Concurrent callers during creation queue on the same promise — so you don't get multiple simultaneous wallet popups. Callers that already grabbed a `SessionKey` reference before refresh continue with the still-valid old key.
3. **DEK cache TTL is set from `ttlMin`.** The DEK cache (a `TtlMap` in `envelope-encryption.ts`, not `dek-manager.ts`) uses the configured `ttlMin` as its TTL. Each entry stamps its own `expiresAt = insertion_time + ttlMin` and is lazily evicted on next access — independent of any specific SessionKey instance. Session-key refresh does **not** invalidate the cache; entries only expire on their own per-entry TTL or via an explicit `client.messaging.encryption.clearCache(groupId?)`.

## Key rotation under load — a Builder gotcha

`client.messaging.rotateEncryptionKey(...)` creates a new `EncryptionHistory` version on-chain ([`client.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/client.ts)). For the combined "remove member + rotate" flow there is also `client.messaging.removeMembersAndRotateKey(...)` — prefer that when revoking access, so the removal and rotation land atomically.

Messages sent **after** rotation use the new `keyVersion`; messages sent before still reference the old version. Old DEKs remain accessible to current group members via Seal — rotation does not break history.

What rotation *does* break: members who lost `MessagingReader` permission **before** rotation can still decrypt history (they have access to old DEK versions). To revoke a member's access to **future** messages, rotate *after* removing their permission (or use `removeMembersAndRotateKey` to do both atomically). See [`docs/sui-stack-messaging/Security.md`](../../../docs/sui-stack-messaging/Security.md) for the full forward-secrecy story.

If you're rotating under load (high message throughput), expect a brief window where in-flight sends may race with the rotation — the SDK handles this internally but adopters should test their dapp's UX through a rotation event.

## Verification

The `SessionKeyManager` ([`encryption/session-key-manager.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/encryption/session-key-manager.ts)) is internal — there is no `client.messaging.encryption.getSessionKey()` on the public API surface. To verify session-key wiring you must trigger an operation that actually uses Seal. The public surface on `client.messaging.encryption` is `encrypt(...)`, `decrypt(...)`, and `clearCache(groupId?)` ([`envelope-encryption.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/encryption/envelope-encryption.ts)).

> **Safety — read before running any of the steps below.**
>
> Every operation in this section that exercises Seal does so by performing a real on-chain or relayer action: `createAndShareGroup` mints a shared group object, `sendMessage` writes ciphertext to the relayer, `rotateEncryptionKey` appends a new `EncryptionHistory` version on-chain (gas-paid by your signer), and `removeMembersAndRotateKey` does both of the above plus removes members. **None of these are reversible.** Rotation in particular cannot be undone — you can only roll forward with another rotation, and every existing group member is affected.
>
> Run these verifications **only against throwaway dev groups on testnet (or localnet)**, never against a production / live group. Concrete recommendation:
>
> - Use a dedicated dev signer with testnet SUI from the faucet.
> - For each verification run, create a **fresh disposable group** (`uuid: \`session-test-${Date.now()}\``) inside the test, and treat it as garbage after. Do not reuse a group across runs.
> - Never call `rotateEncryptionKey` or `removeMembersAndRotateKey` against a group that has real members, real history, or that any user-facing dapp is reading from.
> - If you're integration-testing in a CI environment, scope tests to localnet (`pnpm test:integration` uses a localnet Docker stack — see `ts-sdks/packages/sui-stack-messaging/test/integration/localnet/`) so nothing leaks to a shared network.

With those guardrails in place:

1. **Tier 1 end-to-end** (Node script, dev signer, throwaway group): on a **fresh disposable test group**, call `await client.messaging.createAndShareGroup({ signer, ... })` then `sendMessage`. The first decrypt-needing operation implicitly creates + certifies the session key. No throw → signer reachable + session-key flow works. *Mutating: creates a new on-chain group object and writes one message via the relayer; costs gas.*
2. **Tier 2 sanity** (in your dapp, against a dev group only): trigger any operation that fetches + decrypts an existing message (e.g. `client.messaging.getMessages({ groupRef, limit: 1 })`). This is **read-only** — safe against any group your dev signer can read. Confirm `onSign` is called **once** with a `Uint8Array`; confirm subsequent decryptions within `ttlMin` do **not** re-trigger `onSign`.
3. **TTL behavior** (read-only): set `ttlMin: 1` for a manual test. After ~60s, trigger another `getMessages` call against a dev group (Tier 1: silent refresh; Tier 2: one new `onSign` call). Reset to your production `ttlMin` afterwards. *Non-mutating.*
4. **DEK cache miss on a new key version** (**mutating** — disposable test group only): on a **dev group you have explicitly created for this test**, call `client.messaging.rotateEncryptionKey({...})` ([`client.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/client.ts)). The cache is keyed by `(groupId, keyVersion)`, so the next `getMessages` involving the new version is a cache miss → one fresh Seal round-trip; entries for the old version stay until their per-entry TTL expires. Subsequent fetches for the new version hit the cache. To force-clear cache entries **without mutating chain state** (preferable when iterating), call `client.messaging.encryption.clearCache(groupId?)` — this is the right tool for cache-behavior tests that don't actually need a rotation.

If you're seeing unexpected wallet popups, the most common causes are: (a) `ttlMin` too short, (b) Tier 2 `onSign` is being called for every decryption because the implementation isn't actually returning a valid signature, (c) you're constructing a new client per render (each construction creates its own SessionKeyManager). All three are app-side issues — **don't reach for `rotateEncryptionKey` as a diagnostic step**; it doesn't help with session-key issues and it permanently mutates the group.

## Where this fits

- Bigger integration story: [`integrate-sui-stack-messaging`](../integrate-sui-stack-messaging/SKILL.md).
- Decryption failures and how to diagnose them: [`debug-encryption-flow`](../debug-encryption-flow/SKILL.md).
- Authoritative docs:
  - [`docs/sui-stack-messaging/Setup.md`](../../../docs/sui-stack-messaging/Setup.md) § "Session Key Tiers".
  - [`docs/sui-stack-messaging/Encryption.md`](../../../docs/sui-stack-messaging/Encryption.md) § "Session Key Management".
  - [`docs/sui-stack-messaging/Security.md`](../../../docs/sui-stack-messaging/Security.md) — trust model.
- Implementation: `ts-sdks/packages/sui-stack-messaging/src/encryption/session-key-manager.ts`.
