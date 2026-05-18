---
name: debug-encryption-flow
description: Use when a developer's messages aren't decrypting end-to-end — wrong content, AES-GCM authentication failures, missing DEK, Seal session-key errors, sender-verification failing, or attachments not decrypting. Walks through the envelope-encryption flow stage by stage so the failure can be localized (session key vs DEK vs AAD vs sender verification vs attachment), with the failure modes Builders hit most often. Trigger phrases - "can't decrypt", "decryption fails", "encryption error", "Seal session key error", "wrong message content after decrypt", "AAD mismatch", "AES-GCM authentication failed", "senderVerified false", "attachment decrypt fails", "envelope encryption broken", "key server error".
---

# Debug encryption / decryption failures

When a message can't be read end-to-end, the failure could be in one of five stages. Localizing it is much faster than randomly poking at config. This skill walks through the stages in the order they happen on the receive path, with the most common Builder failure mode for each.

For full context on the encryption model, see [`docs/sui-stack-messaging/Encryption.md`](../../../docs/sui-stack-messaging/Encryption.md) first. This skill is the *diagnosis* counterpart.

## The five stages of receive

```
[Relayer fetch] → [Session key resolve] → [DEK resolve via Seal] → [AES-GCM decrypt with AAD] → [Sender verification]
```

Each can fail independently. Diagnose in this order — earlier failures cascade.

### Stage 1 — Relayer fetch

**Failure surface:** `getMessages` / `getMessage` / `subscribe` throws a `RelayerTransportError` (or your custom transport's error type), or returns zero messages when the group should have some.

**Most common causes:**
- Relayer not reachable / not running. Independent check: `curl <relayerUrl>/health_check`.
- Wrong `groupId` or `relayer.relayerUrl`. Confirm the group exists on the network you're pointed at.
- Relayer's `GROUPS_PACKAGE_ID` doesn't match the network your SDK is on (testnet vs mainnet, or stale localnet ID). The relayer rejects requests for groups it can't find in its membership store.
- (Reference relayer specific) The relayer was restarted and hasn't rebuilt membership from gRPC yet → see the "After relayer restart" troubleshooting note in [`spin-up-e2e-stack`](../spin-up-e2e-stack/SKILL.md).

If `getMessages` returns ciphertext but `decrypt` fails downstream, this stage is fine — move on.

### Stage 2 — Session key resolve

**Failure surface:** Errors from `@mysten/seal` like "session key expired", "session key not certified", or wallet popup for personal-message signing fails / hangs.

**Most common causes (by tier):**
- **Tier 1 (`{ signer }`)**: signer is misconfigured (wrong network, no `toSuiAddress`); a fresh signer is created on every render so the session key never has time to certify.
- **Tier 2 (`{ address, onSign }`)**: `onSign` returns the wrong type (must be the signature string from `signPersonalMessage`), throws silently, or the wallet adapter is on a different network than the SDK.
- **Tier 3 (`{ getSessionKey }`)**: the returned key isn't certified, or has already expired.

Diagnostic: `SessionKeyManager` is internal (no public `getSessionKey()` on `client.messaging.encryption`). Trigger any decrypt-needing operation (e.g. `client.messaging.getMessages({ groupRef, limit: 1 })`) and check the thrown error: if it's a Seal-side error mentioning session/certification, the failure is here. If decryption is needed and *no* wallet popup appears at all (Tier 2), the SDK isn't reaching `SessionKeyManager.create` — usually a wiring/config issue (e.g. encryption config missing entirely).

See [`configure-session-keys`](../configure-session-keys/SKILL.md) for full lifecycle.

### Stage 3 — DEK resolve via Seal

**Failure surface:** Errors mentioning "Seal decryption failed", "threshold not met", "seal_approve denied", or `keyVersion` not found.

**Most common causes:**
- **Wrong Seal `serverConfigs`** — the `objectId`s don't exist or are deprecated; threshold doesn't match the count of key servers; the package the SDK is on is different from the package the key servers are configured for. Confirm against Seal docs and `docs/sui-stack-messaging/Setup.md`.
- **Caller doesn't have `MessagingReader` permission** on the group. `seal_approve_reader` checks group membership — if you can't decrypt and your address was just removed from the group, that's expected behavior. If you should have access, check the on-chain `PermissionedGroup<Messaging>` object's members list.
- **`keyVersion` references a version that doesn't exist** in `EncryptionHistory` — usually means the message was sent against a different group than the one the SDK is now querying (wrong `groupId` / UUID derivation collision).
- **Custom Seal policy rejects the call** (if you have one) — your `seal_approve` function's assertions are failing. Test the policy independently with `extend-smart-contracts` recipes.

Diagnostic: instrument `client.messaging.encryption.decrypt(...)` with try/catch and log the inner error from Seal — it usually says exactly which key server returned which status code.

### Stage 4 — AES-GCM decrypt with AAD

**Failure surface:** "AES-GCM authentication failed" / "OperationError" from Web Crypto, or you get plaintext-shaped bytes but they decode to garbage.

The AAD format binds ciphertext to context. It's a BCS struct (`MessageAAD = bcs.struct({ groupId: bcs.Address, keyVersion: bcs.u64(), senderAddress: bcs.Address })`) — see `buildMessageAad` in [`envelope-encryption.ts`](../../../ts-sdks/packages/sui-stack-messaging/src/encryption/envelope-encryption.ts). The BCS serialization of three fixed-size primitives concatenates to:

```
[groupId (32 bytes)][keyVersion (8 bytes LE u64)][senderAddress (32 bytes)]  // 72 bytes total
```

If *any* of those three fields doesn't match what the sender used at encrypt time, AES-GCM **rejects the ciphertext entirely** — that's by design, it's an authentication failure, not a decryption-with-wrong-output.

**Most common causes:**
- **`senderAddress` mismatch** — the message's `senderAddress` in the relayer payload doesn't match what's used for AAD reconstruction. This is rare with the default relayer; can happen if a custom `RelayerTransport` mutates the field. Confirm with [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md) verification step.
- **`keyVersion` mismatch** — the message says it was encrypted under version N but the relayer is delivering it labeled as version M. This is usually a bug in a custom backend that doesn't preserve `keyVersion` round-trip.
- **`groupId` mismatch** — almost always a wrong-group-fetch bug; the message belongs to a different group than the one the SDK is decrypting for. Check your `GroupRef` derivation (UUIDs derive `groupId` deterministically; explicit IDs must match exactly).
- **The DEK is correct but for the wrong version of the message** (edit/delete history) — if the message was edited, decryption uses the *new* ciphertext + nonce; old ciphertext won't decrypt under the new AAD.

If you got past Stage 3 but fail here, you have a context mismatch — log all three AAD components on both sides.

### Stage 5 — Sender verification

**Failure surface:** `DecryptedMessage.senderVerified === false` (no exception thrown — the message decrypts successfully but the signature check failed).

The per-message signature is over:
```
"{groupId}:{hex(encryptedText)}:{hex(nonce)}:{keyVersion}"
```

**Most common causes:**
- **Custom relayer stripped or rewrote `signature` / `publicKey`** on the `RelayerMessage`. See [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md) — these fields must round-trip losslessly.
- **Relayer is impersonating senders** (or there's a real bug in your custom relayer) — `publicKey` doesn't derive to `senderAddress`.
- **Sender used a wallet with a scheme the SDK doesn't yet support** — the SDK supports Ed25519, Secp256k1, Secp256r1; zkLogin signatures are *not yet supported* for sender verification (see [GitHub issue #63](https://github.com/MystenLabs/sui-stack-messaging/issues/63)). If your sender used a wallet with a non-Ed25519/Secp* scheme, `senderVerified` will be false.

If decryption itself succeeds and only `senderVerified` is false, the *contents* are still trustworthy if you trust your relayer — but the SDK is correctly flagging that it can't independently verify authorship. Surface this in your UI as a warning.

## Attachment-specific issues

Attachments use a separate encryption envelope per attachment (same DEK, different nonce; metadata encrypted separately).

If text decrypts fine but attachments don't:
- Confirm `attachments.storageAdapter` is configured — without it, attachment metadata is decryptable but `getAttachmentData` won't work.
- Check the storage adapter — for `WalrusHttpStorageAdapter`, hit the aggregator URL directly with the patch ID to confirm the bytes are reachable.
- File metadata (fileName / mimeType / fileSize) is encrypted with its own nonce; if that decrypts but the file body doesn't, the issue is in the storage adapter's download, not encryption.

See [`docs/sui-stack-messaging/Attachments.md`](../../../docs/sui-stack-messaging/Attachments.md) for the full attachment encryption model.

## Quick-start instrumentation

There's no public way to probe the session-key stage in isolation (`SessionKeyManager` is internal). The cheapest diagnostic is a single `getMessages` call that exercises Stages 1 → 5, with error-type discrimination:

```ts
import { RelayerTransportError } from '@mysten/sui-stack-messaging';

try {
  const { messages } = await client.messaging.getMessages({ groupRef, limit: 1 });
  console.log('Stages 1-4 OK:', messages[0]);
  console.log('Stage 5 senderVerified:', messages[0]?.senderVerified);
} catch (err) {
  if (err instanceof RelayerTransportError) {
    console.error('Stage 1 (relayer):', err.status, err.code, err.message);
  } else if (err instanceof Error && /session|certif|seal/i.test(err.message)) {
    console.error('Stage 2 (session key) or Stage 3 (Seal/DEK):', err);
  } else if (err instanceof Error && /OperationError|authentication/i.test(err.message)) {
    console.error('Stage 4 (AES-GCM AAD mismatch):', err);
  } else {
    console.error('Unknown failure:', err);
  }
}
```

If `senderVerified` is `false` but no exception was thrown → Stage 5 alone. If you only want to probe Stages 1-3 (no encrypted content to decrypt), `client.messaging.transport.fetchMessages(...)` returns the raw `RelayerMessage` without attempting decryption — useful for isolating whether your relayer/auth wiring works at all before debugging encryption.

## Where this fits

- Encryption model deep-dive: [`docs/sui-stack-messaging/Encryption.md`](../../../docs/sui-stack-messaging/Encryption.md).
- Trust model: [`docs/sui-stack-messaging/Security.md`](../../../docs/sui-stack-messaging/Security.md).
- Implementation references:
  - `ts-sdks/packages/sui-stack-messaging/src/encryption/envelope-encryption.ts` — main encrypt/decrypt path.
  - `ts-sdks/packages/sui-stack-messaging/src/encryption/dek-manager.ts` — DEK cache.
  - `ts-sdks/packages/sui-stack-messaging/src/encryption/session-key-manager.ts` — session key lifecycle.
  - `ts-sdks/packages/sui-stack-messaging/src/verification.ts` — sender verification.
- Related skills: [`configure-session-keys`](../configure-session-keys/SKILL.md), [`configure-custom-relayer-transport`](../configure-custom-relayer-transport/SKILL.md), [`extend-smart-contracts`](../extend-smart-contracts/SKILL.md) (for custom Seal policies).
