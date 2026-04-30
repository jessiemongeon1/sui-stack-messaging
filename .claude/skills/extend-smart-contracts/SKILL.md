---
name: extend-smart-contracts
description: Use when the user wants to add custom Move modules on top of sui_stack_messaging — custom seal policies (subscription-based, token-gated), paid-join rules, custom permission types, or any extension package depending on the canonical messaging contracts. Trigger phrases - "custom seal policy", "paid join rule", "extend the messaging contracts", "add a Move module", "custom permission type", "token-gated messaging", "subscription messaging", "extend smart contracts".
---

# Extend the smart contracts

`move/packages/sui_stack_messaging/` is canonical and already published. **Do not fork it.** To add custom behavior, write your own Move package that depends on it. The repo ships two worked examples in `move/packages/example_app/`.

This skill covers only what's specific to extending `sui_stack_messaging`. For general Move-language quality checks, the community-maintained [move-code-quality-skill](https://github.com/1NickPappas/move-code-quality-skill) is one option.

For canonical mainnet/testnet package IDs (`sui_stack_messaging` and `sui_groups`), see the "Canonical package addresses" section in the repo-root [CLAUDE.md](../../../CLAUDE.md) / [AGENTS.md](../../../AGENTS.md).

## Package layout you'll be working against

```
move/packages/
├── sui_stack_messaging/         CANONICAL — depend on, don't modify
│   └── sources/
│       ├── messaging.move           public entry; defines permission types
│       ├── seal_policies.move       seal_approve_reader entry; access control
│       ├── encryption_history.move  key versioning
│       ├── group_leaver.move        self-service leave
│       ├── group_manager.move       SuiNS / metadata admin
│       ├── metadata.move            VecMap key-value group metadata
│       └── version.move             package version gating
└── example_app/                 REFERENCE — copy-paste starting point
    └── sources/
        ├── custom_seal_policy.move  subscription-based reader policy
        └── paid_join_rule.move      payment-gated MessagingReader grant
```

The messaging package depends on `sui-groups` and operates on its types directly. Types like `PermissionedGroup<T>` and `PermissionsAdmin` come from `sui_groups::permissioned_group` and your extension package will `use` them from there too. At runtime, `messaging.move` creates `PermissionedGroup<Messaging>` instances via `permissioned_group::new_derived(...)`. The package adds its own messaging-specific permission witness types: `MessagingSender`, `MessagingReader`, `MessagingEditor`, `MessagingDeleter`, `MetadataAdmin`, `SuiNsAdmin`.

## Skeleton: a new extension package

Scaffold a fresh Move package with the Sui CLI, then add the messaging dependencies:

```bash
sui move new my_messaging_extension
cd my_messaging_extension
```

This creates `Move.toml`, `sources/`, and `tests/` with sensible defaults. Edit `Move.toml` to add the dependencies:

```toml
[package]
name = "my_messaging_extension"
edition = "2024"

[dependencies]
sui_stack_messaging = { git = "https://github.com/MystenLabs/sui-stack-messaging.git", subdir = "move/packages/sui_stack_messaging", rev = "main" }
sui_groups          = { git = "https://github.com/MystenLabs/sui-groups.git", subdir = "move/packages/sui_groups", rev = "ea766818b90e162341e885a855718388edcc8e99" } # tag mainnet/v1
```

Pin a release tag instead of `main` for reproducible builds. The `Sui` framework dependency is added automatically by `sui move new`.

## Extension pattern 1 — custom Seal policy

Use case: "only paid subscribers can read this group's messages."

Pattern: declare an `entry fun seal_approve` (visibility is just `entry`, not `public entry`) in your package. Seal key servers call this function via dry-run during decryption. It takes Seal-injected `id: vector<u8>` first, your custom proof objects, then the canonical messaging objects (`PermissionedGroup<Messaging>`, `EncryptionHistory`), `Clock`, and `&TxContext`. Always start the body with `sui_stack_messaging::seal_policies::validate_identity(group, encryption_history, id)` to enforce the standard identity-bytes contract, then add your custom checks.

References:

- Move source: `move/packages/example_app/sources/custom_seal_policy.move`.
- Design doc: `move/design_docs/example_app/custom_seal_policy.md`.
- Working integration test (full Move + TS flow against localnet): `ts-sdks/packages/sui-stack-messaging/test/integration/localnet/custom-seal-policy.test.ts`.

Shape (mirrors the example):

```move
module my_ext::token_gated_policy;

use sui_groups::permissioned_group::PermissionedGroup;
use sui_stack_messaging::messaging::Messaging;
use sui_stack_messaging::encryption_history::EncryptionHistory;
use sui_stack_messaging::seal_policies;
use sui::clock::Clock;

const ENoAccess: u64 = 1;

entry fun seal_approve_subscription<Token: drop>(
    id: vector<u8>,
    sub: &Subscription<Token>,                      // your custom proof
    service: &Service<Token>,                       // your custom context
    group: &PermissionedGroup<Messaging>,
    encryption_history: &EncryptionHistory,
    clock: &Clock,
    ctx: &TxContext,
) {
    // 1. Reuse standard identity validation:
    //    - id parses as [group_id (32 bytes)][key_version (8 bytes LE u64)]
    //    - group_id in id matches `group`
    //    - encryption_history belongs to `group`
    //    - key_version exists in encryption_history
    seal_policies::validate_identity(group, encryption_history, id);

    // 2. Your custom checks (membership, subscription validity, etc.).
    assert!(check_policy(sub, service, group, clock, ctx), ENoAccess);
}
```

Notes:

- Visibility is `entry` (no `public`)
- Identity bytes are the standard format `[group_id (32)][key_version (8 LE u64)]`, produced by the SDK. Don't invent a custom layout — `validate_identity` enforces it.

### Wire it client-side via `SealPolicy`

Custom `seal_approve` functions are wired to the SDK through the `SealPolicy<TApproveContext>` interface — **not** as a `$extend()` extension. Pass an instance under `encryption.sealPolicy` at client creation; otherwise `DefaultSealPolicy` (which targets the canonical `seal_policies::seal_approve_reader`) is used.

The interface has two members (`ts-sdks/packages/sui-stack-messaging/src/encryption/seal-policy.ts`):

- `readonly packageId: string` — your extension package's original (V1) package ID. Becomes the Seal encryption namespace (instead of the messaging package's).
- `sealApproveThunk(identityBytes, groupId, encryptionHistoryId, ...context)` — returns a `(tx) => tx.moveCall({...})` thunk. Seal's key servers dry-run the resulting tx during decryption to authorize access.

The optional `TApproveContext` generic carries any extra runtime IDs your `seal_approve` needs (e.g., subscription / service / NFT). When set, `approveContext` becomes a required parameter on `sendMessage`, `getMessages`, etc., and the SDK threads it through to your thunk.

Sketch:

```ts
import type { SealPolicy } from "@mysten/sui-stack-messaging";
import type { Transaction, TransactionResult } from "@mysten/sui/transactions";

interface SubContext {
  serviceId: string;
  subscriptionId: string;
}

class SubscriptionSealPolicy implements SealPolicy<SubContext> {
  readonly packageId = MY_EXT_PACKAGE_ID;

  sealApproveThunk(
    identityBytes,
    groupId,
    encryptionHistoryId,
    context: SubContext,
  ) {
    return (tx: Transaction): TransactionResult =>
      tx.moveCall({
        target: `${MY_EXT_PACKAGE_ID}::token_gated_policy::seal_approve`,
        typeArguments: ["0x2::sui::SUI"],
        arguments: [
          tx.pure.vector("u8", identityBytes),
          tx.object(context.subscriptionId),
          tx.object(context.serviceId),
          tx.object(groupId),
          tx.object(encryptionHistoryId),
          tx.object("0x6"), // Clock
        ],
      });
  }
}

const client = createSuiStackMessagingClient<SubContext>(baseClient, {
  encryption: {
    sessionKey: { signer },
    sealPolicy: new SubscriptionSealPolicy(),
  },
  relayer: { relayerUrl: "..." },
});

await client.messaging.sendMessage({
  signer,
  groupRef: { uuid: "my-group" },
  text: "Hello!",
  approveContext: { serviceId: "0x...", subscriptionId: "0x..." },
});
```

References:

- Full walk-through with both Move + TS sides: `docs/sui-stack-messaging/Extending.md` ("Custom Seal Policy" section).
- Interface definition + JSDoc: `ts-sdks/packages/sui-stack-messaging/src/encryption/seal-policy.ts`.
- Default implementation as a worked example: `DefaultSealPolicy` in the same file.
- Where the policy is plugged into the encryption pipeline: `ts-sdks/packages/sui-stack-messaging/src/encryption/envelope-encryption.ts`.

## Extension pattern 2 — gated membership (Actor object pattern)

Use case: "users pay SUI to self-serve join the group."

Pattern: the **actor object pattern** — a shared object that holds delegated permissions on the group and exposes restricted operations to arbitrary callers. The actor's `&UID` is the authority token: the group's `object_grant_permission` API checks that the holder of that UID has the right permission, then performs the grant on the actor's behalf. End users never hold admin permissions directly; they interact with the actor.

Setup flow (mirrors `paid_join_rule.move`):

1. Admin creates a group via `messaging::create_group(...)`.
2. Admin creates the actor object (e.g. `PaidJoinRule<Token>`) — a `has key` object with a UID and any state it needs (fee, accumulated balance, etc.).
3. Admin grants `ExtensionPermissionsAdmin` to the actor's address:
   `group.grant_permission<Messaging, ExtensionPermissionsAdmin>(rule_address, ctx)`.
4. Actor is `transfer::share_object`'d so anyone can use it.
5. The actor's public `join` function takes payment, then grants `MessagingReader` (membership) to the sender by passing its own `&UID` as authority:
   `group.object_grant_permission<Messaging, MessagingReader>(&rule.id, ctx.sender())`.

Actor objects also appear inside the canonical messaging package itself: `group_leaver` (self-service leave without admin permission) and `group_manager` (SuiNS reverse lookup + metadata admin) follow the same shape.

References:

- Worked example: `move/packages/example_app/sources/paid_join_rule.move`.
- Design doc for this example: `move/design_docs/example_app/paid_join_rule.md`.
- Working integration test (full grant-permissions setup, payment, member-add flow against localnet): `ts-sdks/packages/sui-stack-messaging/test/integration/localnet/paid-join-rule.test.ts`.
- Pattern overview, including the canonical actors: `move/design_docs/REQUIREMENTS.md`.
- Canonical actors: `move/design_docs/sui_stack_messaging/group_leaver.md`, `move/design_docs/sui_stack_messaging/group_manager.md`.

Note:

You might want to implement a more comprehensive payment system than what the `paid_join_rule.move` example does.
`sui-payment-kit` might interest you: https://github.com/MystenLabs/sui-payment-kit

## Extension pattern 3 — custom permission type

Use case: "I want a `Moderator` role that can delete but not edit."

Before writing any Move, ask whether you actually need a _new_ permission witness. The four canonical messaging permissions (`MessagingSender`, `MessagingReader`, `MessagingEditor`, `MessagingDeleter`) compose freely — a "moderator who can delete but not edit" is just `MessagingDeleter` granted without `MessagingEditor`. That requires no new Move at all; you grant subsets via the `@mysten/sui-groups` TS SDK.

If you do need a brand new permission type, the work splits across three layers:

### 1. Declare the witness type (Move, minimal)

A permission witness is just a phantom marker struct. It must live in a published Move module — types can't be invented at runtime — but the module can be a one-liner in your extension package:

```move
module my_ext::roles;

public struct Moderator() has drop;
```

Publish with `sui client publish`. Note the package ID; that's all the on-chain footprint you need.

### 2. Grant / revoke via the sui-groups TS SDK

You do **not** need custom Move for granting. The canonical `permissioned_group` module is exposed by the `@mysten/sui-groups` extension, which is already chained on the messaging client (`suiStackMessaging` requires `suiGroups` upstream). The chain is:

```ts
const client = new SuiClient({ url })
  .$extend(
    suiGroups({ witnessType: `${MESSAGING_PKG}::messaging::Messaging` }),
    seal({
      /* ... */
    }),
  )
  .$extend(
    suiStackMessaging({
      /* ... */
    }),
  );

// Now both namespaces are available on the same client:
//   client.groups   — sui-groups SDK (permissions, members)
//   client.messaging — sui-stack-messaging SDK
```

The witness type is fixed at extension creation, so `client.groups` is bound to `Messaging`. Per-call options carry only the _permission_ type. To grant `Moderator`:

```ts
import { Transaction } from "@mysten/sui/transactions";

const tx = new Transaction();
tx.add(
  client.groups.call.grantPermission({
    groupId,
    member: memberAddr,
    permissionType: `${MY_EXT_PKG}::roles::Moderator`,
  }),
);
// ...sign and submit tx with the admin's signer.
```

The convenience layer `client.groups.tx.grantPermission({ transaction, ... })` and the top-level `await client.groups.grantPermission({ signer, ... })` (build + sign + submit) are also available — see `~/Dev/work/projects/sui-groups/ts-sdks/packages/sui-groups/src/{call,transactions,client}.ts`.

The caller of this tx must hold `PermissionsAdmin` on the group (the creator does by default).

In Move, the equivalent is `group.grant_permission<Messaging, Moderator>(member_addr, ctx)` — the same method the `paid_join_rule` example uses to grant `FundsManager` (`paid_join_rule.move:188`).

### 3. Enforce the permission

This is the part that's easy to miss. Granting a custom permission stores it on-chain, but it doesn't gate anything by itself.

- **Off-chain enforcement (relayer)** — the reference relayer hard-codes the four canonical permissions in `relayer/src/auth/permissions.rs` and maps HTTP methods to them in `relayer/src/auth/middleware.rs:80–83`. A custom permission like `Moderator` is **not** recognized today; the relayer would either ignore it or reject the request depending on the operation. To enforce custom permissions on relayer endpoints, fork the relayer to extend the `MessagingPermission` enum and the method-to-permission mapping. See [`develop-relayer`](../develop-relayer/SKILL.md).
- **On-chain enforcement** — only relevant if your custom permission gates Move-level behavior (e.g., a Move function in your extension package that should only run for moderators). Check with `group.has_permission<Messaging, Moderator>(ctx.sender())` from `sui-groups`, exactly like the `paid_join_rule` example checks `FundsManager` before withdrawal (`paid_join_rule.move:266`).

The canonical messaging permissions remain untouched in either case — your custom type lives alongside them.

## Build & test

```bash
# Build messaging (canonical) — usually only needed for codegen
sui move build --path move/packages/sui_stack_messaging
sui move test  --path move/packages/sui_stack_messaging

# Build & test the example extension package
sui move build --path move/packages/example_app
sui move test  --path move/packages/example_app

# Build & test your own package (same flags, your path)
sui move build --path /path/to/my_messaging_extension
sui move test  --path /path/to/my_messaging_extension
```

Move edition: `2024`.

For a broader index of runnable usage examples (the full integration test suite, plus baseline / view / metadata / archive flows), see the "Runnable usage examples" section in the parent skill: [`develop-on-sui-stack-messaging`](../develop-on-sui-stack-messaging/SKILL.md).

## Publish

Publish your extension package with the Sui CLI like any other Move package:

```bash
sui client switch --env testnet                       # or mainnet
sui client active-address                             # confirm deployer address has gas
sui client publish --gas-budget 200000000 /path/to/my_messaging_extension
```

Capture the printed package ID from the transaction effects — you'll wire it into your app config. The CLI writes the deployed address into `Move.lock` (and `Published.toml` if you opt into automated address management; see Sui docs on `sui move manage-package`).

The `publish/` directory in this repo is a maintainer-only helper used to publish the canonical `sui_stack_messaging` package. Do not use it for your extension package — use `sui client publish` directly or your own custom publishing scripts.

## After publishing — wire to the SDK

The TS SDK is composed via the Sui client extension system (`$extend()`). The recommended pattern is to define your own extension on top of `suiStackMessaging`, mirroring how `suiStackMessaging` itself sits on top of `suiGroups` + `seal`.

```ts
// In your lightweight SDK: ship a single extension factory.
// my-messaging-ext-sdk/src/index.ts

import type { ClientWithExtensions } from '@mysten/sui/client';
import type { SuiStackMessagingClient } from '@mysten/sui-stack-messaging';

export function myMessagingExt(options: { packageId: string }) {
  return {
    name: 'myMessagingExt' as const,
    register: (client: ClientWithExtensions<{ messaging: SuiStackMessagingClient }>) => ({
      // Build txs that call your custom Move entry functions, reusing the
      // messaging client for things like envelope encryption / relayer access.
      paidJoin: async (groupUuid: string, payment: /* ... */) => { /* ... */ },
    }),
  };
}
```

Consumers then chain it after `suiStackMessaging`:

```ts
const client = new SuiGrpcClient({ network: "testnet" })
  .$extend(
    suiGroups({ witnessType: `${pkg}::messaging::Messaging` }),
    seal({
      /* ... */
    }),
  )
  .$extend(
    suiStackMessaging({
      /* ... */
    }),
  )
  .$extend(myMessagingExt({ packageId: MY_EXT_PACKAGE_ID }));

await client.myMessagingExt.paidJoin(groupUuid, payment);
await client.messaging.sendMessage({
  /* ... */
}); // canonical SDK still available
```

Why this beats a fork:

- You inherit the messaging client's encryption, relayer transport, and recovery flows for free.
- Consumers compose your extension with any future canonical SDK release without re-vendoring.
- The canonical SDK auto-detects `sui_stack_messaging` package IDs on testnet/mainnet (`TESTNET_SUI_STACK_MESSAGING_PACKAGE_CONFIG` / `MAINNET_SUI_STACK_MESSAGING_PACKAGE_CONFIG` in `ts-sdks/packages/sui-stack-messaging/src/constants.ts`) — your extension only needs to know its own package ID.

For trivial cases (a single tx call from app code), you can skip the extension and just build the transaction inline with `Transaction` from `@mysten/sui/transactions`.

References:

- Client extension system reference: `docs/sui-stack-messaging/Setup.md` ("Manual setup" section).
- The `suiStackMessaging` extension itself is a worked example of the pattern: `ts-sdks/packages/sui-stack-messaging/src/client.ts`.
- Mysten SDK building guidelines: https://sdk.mystenlabs.com/sui/sdk-building (linked from the root README).

## What NOT to do

- Do not fork `sui_stack_messaging` to add a new policy. Build a new package that depends on it. Forking puts you on a separate upgrade path from canonical.
- Do not redefine messaging permission types (`MessagingSender`, etc.) in your package. Add new types alongside.
- Do not edit `Published.toml` for the canonical package. It is committed and load-bearing for upgrade paths.

## Cross-links

- Extending guide (TS side, with Move snippets): `docs/sui-stack-messaging/Extending.md`.
- Move package architecture: `move/design_docs/REQUIREMENTS.md`.
- Per-module design docs: `move/design_docs/sui_stack_messaging/{messaging,seal_policies,encryption_history,group_leaver,group_manager,metadata,version}.md`.
- Worked examples: `move/design_docs/example_app/{custom_seal_policy,paid_join_rule}.md`.
- General Move quality check (community-maintained, optional): https://github.com/1NickPappas/move-code-quality-skill.
