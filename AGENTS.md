# AGENTS.md

Guidance for AI coding agents (Cursor, Copilot, Codex, Aider, OpenAI agents, and similar) working in this repository. Equivalent guidance to `CLAUDE.md`, agent-neutral phrasing — the two files cover the same repo invariants and commands but differ in the "skills" sections (CLAUDE.md describes the in-repo Claude Code skills; this file points at the SKILL.md files as plain Markdown for any agent to read).

## What this repo is

Sui Stack Messaging — encrypted messaging tooling for Web3 apps, built on Sui, Seal, and Walrus. Multi-component repo: canonical Move contracts and TypeScript SDK, plus reference implementations of a relayer, indexer, and demo chat UI that downstream projects will fork.

## Repo layout

```
move/                                       CANONICAL  Move smart contracts
  packages/sui_stack_messaging/               canonical messaging package (published mainnet+testnet)
  packages/example_app/                       reference extension examples (custom seal policy, paid join rule)
  design_docs/                                Move-side design docs

ts-sdks/packages/sui-stack-messaging/        CANONICAL  TypeScript SDK (@mysten/sui-stack-messaging on npm)

relayer/                                    REFERENCE  Rust axum relayer (fork-and-extend)
walrus-discovery-indexer/                   REFERENCE  TS indexer for Walrus blobs (fork-and-extend)
chat-app/                                   REFERENCE  Vite + React 19 demo (fork-and-extend)

publish/                                    INTERNAL   maintainer-only helper for publishing the canonical Move package
docs/sui-stack-messaging/                   Authoritative dev documentation
```

External canonical dependency: [`sui-groups`](https://github.com/MystenLabs/sui-groups). Treat as published infrastructure; do not vendor.

## Canonical vs reference — the mental model

**Canonical** (`move/packages/sui_stack_messaging/`, `ts-sdks/packages/sui-stack-messaging/`): consume; do not fork.
- Move contracts are already published to mainnet and testnet (see "Canonical package addresses" below).
- The SDK is on npm as `@mysten/sui-stack-messaging`. Pin the version, don't vendor the source.
- To extend: write your own Move package depending on `sui_stack_messaging` (see `move/packages/example_app/` for two worked examples), or implement the SDK's transport/storage/recovery interfaces.

**Reference** (`relayer/`, `walrus-discovery-indexer/`, `chat-app/`): fork as a starting point and modify freely.
- Intentionally minimal. Real deployments will replace storage backends, auth strategies, observability, deployment topology.
- The SDK talks to the relayer through the `RelayerTransport` TS interface — implement your own transport without forking the Rust code, or fork the Rust code.

## Build / test / lint per package

### Move (`move/packages/sui_stack_messaging/`, `move/packages/example_app/`)

```bash
sui move build --path move/packages/sui_stack_messaging
sui move test  --path move/packages/sui_stack_messaging
sui move build --path move/packages/example_app
sui move test  --path move/packages/example_app
```

Edition: `2024`. Sui CLI toolchain pinned to `1.68.1` for canonical builds (`Published.toml`).

### TypeScript SDK (`ts-sdks/`)

From `ts-sdks/`:

```bash
pnpm install                      # pnpm >=10.17.0
pnpm build                        # turbo run build
pnpm test                         # turbo run test
pnpm lint                         # oxlint + prettier
pnpm lint:fix
```

From `ts-sdks/packages/sui-stack-messaging/`:

```bash
pnpm build                        # tsc --noEmit + tsdown
pnpm test                         # typecheck + unit
pnpm test:unit                    # vitest run unit
pnpm test:integration             # vitest with localnet config
pnpm test:e2e                     # vitest e2e against testnet
pnpm codegen                      # regenerate Move bindings
```

### Relayer (`relayer/`)

```bash
cargo run                         # dev server :3000 (after cp .env.example .env, set GROUPS_PACKAGE_ID)
cargo test
cargo test -- --ignored           # tests that hit Walrus testnet
cargo fmt
cargo clippy --all-targets -- -D warnings
docker compose up                 # alternative: containerized
```

Toolchain: stable Rust, components `clippy` + `rustfmt` (`relayer/rust-toolchain.toml`).

### Walrus discovery indexer (`walrus-discovery-indexer/`)

```bash
pnpm install
pnpm dev                          # tsx watch
pnpm build                        # tsc -> dist/
pnpm start                        # node dist/index.js
pnpm test                         # vitest run
```

### Chat-app (`chat-app/`)

```bash
pnpm install
pnpm build:deps                   # builds the linked SDK from ts-sdks/
pnpm dev                          # vite :5173
pnpm build                        # tsc -b && vite build
pnpm preview
```

The chat-app links the SDK locally: `"@mysten/sui-stack-messaging": "link:../ts-sdks/packages/sui-stack-messaging"`. Re-run `pnpm build:deps` after SDK changes.

## Repo invariants

- **Do not edit `ts-sdks/packages/sui-stack-messaging/src/contracts/sui_stack_messaging/`** — auto-generated by `pnpm codegen`.
- **Do not edit `move/packages/sui_stack_messaging/Published.toml`** — committed source of truth for deployed package IDs.
- **Do not fork the canonical Move package** — to extend, write your own package depending on `sui_stack_messaging`.
- **Do not redefine canonical permission types** (`MessagingSender`, `MessagingReader`, `MessagingEditor`, `MessagingDeleter`, `MetadataAdmin`, `SuiNsAdmin`).
- **Do not break the relayer wire protocol** — SDK and indexer depend on its message and Walrus archive formats.
- **Do not bump SDK versions ad hoc** — coordinate via `pnpm changeset` (see `ts-sdks/RELEASING.md`).
- **`minimumReleaseAge: 2880`** is enforced by `ts-sdks/pnpm-workspace.yaml` (2-day floor on new dependency releases). Mysten packages are excluded.

## Canonical package addresses

From `move/packages/sui_stack_messaging/Published.toml`:

| Network | sui_stack_messaging |
|---|---|
| Mainnet | `0xcbd2f4c25c7f799c45c0c9f221850178b711b2c89916c8e99038aa8ac609a62e` |
| Testnet | `0x047696be0e98f1b47a99727fecf2955cadb23c56f67c6b872b74e3ad59d51b46` |

Sui Groups (from `relayer/.env.example`):

| Network | sui_groups |
|---|---|
| Mainnet | `0x541840ae7df705d1c6329c22415ed61f9140a18b79b13c1c9dc7415b115c1ba8` |
| Testnet | `0xba8a26d42bc8b5e5caf4dac2a0f7544128d5dd9b4614af88eec1311ade11de79` |

The SDK auto-detects testnet/mainnet via constants in `ts-sdks/packages/sui-stack-messaging/src/constants.ts`.

## Where to find documentation

Authoritative docs live in `docs/sui-stack-messaging/`:

- `Installation.md`, `Setup.md`, `Examples.md` — getting started.
- `APIRef.md` — full SDK API reference.
- `Encryption.md`, `Security.md` — crypto + trust model. Read before changing auth or transport.
- `Relayer.md` — protocol the relayer implements (required reading before forking it).
- `Attachments.md`, `ArchiveRecovery.md`, `GroupDiscovery.md` — feature deep dives.
- `Extending.md` — custom seal policies, transports, storage adapters, recovery transports.
- `Testing.md` — test strategy across SDK / relayer / Move.

Per-component:

- `relayer/README.md` — relayer reference (auth pipeline, API, storage, sync).
- `walrus-discovery-indexer/README.md` + `walrus-discovery-indexer/docs/README.md` — indexer reference.
- `chat-app/README.md` + `chat-app/docs/SYSTEM_DESIGN.md` — demo app reference.
- `move/design_docs/REQUIREMENTS.md` + `move/design_docs/sui_stack_messaging/*.md` — Move package design.
- `ts-sdks/RELEASING.md` — release/changeset workflow.

## Skills folder for Claude Code agents

`.claude/skills/` contains skill files used by Claude Code. Other agents can read them as plain Markdown — each `SKILL.md` is a focused, command-first runbook for one task:

- `develop-on-sui-stack-messaging/SKILL.md` — orientation; canonical vs reference.
- `spin-up-relayer/SKILL.md` — run the reference relayer locally.
- `spin-up-e2e-stack/SKILL.md` — relayer + indexer + chat-app together.
- `develop-relayer/SKILL.md` — fork-and-extend the Rust relayer.
- `develop-walrus-indexer/SKILL.md` — fork-and-extend the TS indexer.
- `extend-smart-contracts/SKILL.md` — add custom Move modules on top of `sui_stack_messaging`.

## Tooling baseline

- Node: pnpm `>=10.17.0`.
- Rust: stable + clippy + rustfmt.
- Sui CLI: required for Move build/test/publish.
- Move edition: 2024.

## Code style

- **TypeScript** (`ts-sdks/`): formatted by Prettier, linted by oxlint. Both configured at `ts-sdks/` root (`prettier.config.js`, oxlint via `pnpm oxlint:check`). Run `pnpm lint:fix` to auto-format. Don't introduce ESLint or other linters — oxlint is the chosen linter here.
- **Rust** (`relayer/`): formatted by `rustfmt`, linted by `clippy` (toolchain pinned in `relayer/rust-toolchain.toml`). Run `cargo fmt` and `cargo clippy --all-targets -- -D warnings` before declaring done.
- **Move** (`move/`): edition `2024`. No formatter shipped in-repo.
- **Imports**: TS uses `@ianvs/prettier-plugin-sort-imports`; don't reorder by hand.
- **No emojis** in committed code, comments, or docs unless explicitly requested.
- **Comments**: write only when the *why* is non-obvious. Don't restate what the code does or reference the current task / PR / issue number.

## Programmatic checks — run before declaring a task done

Match the checks to the package(s) you touched. If you touched several, run all relevant sets.

```bash
# TypeScript SDK changes (anywhere under ts-sdks/)
cd ts-sdks
pnpm lint                          # oxlint + prettier check
pnpm build                         # turbo run build
pnpm --filter @mysten/sui-stack-messaging test:unit
pnpm --filter @mysten/sui-stack-messaging test:typecheck

# Move changes (anywhere under move/)
sui move build --path move/packages/sui_stack_messaging
sui move test  --path move/packages/sui_stack_messaging
sui move build --path move/packages/example_app
sui move test  --path move/packages/example_app
# If Move public API changed, also regenerate SDK bindings:
cd ts-sdks/packages/sui-stack-messaging && pnpm codegen

# Relayer changes
cd relayer
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test                         # network-free tests

# Walrus indexer changes
cd walrus-discovery-indexer
pnpm build                         # tsc (typecheck + emit)
pnpm test
```

Heavier suites (`pnpm test:integration`, `pnpm test:e2e`, `cargo test -- --ignored`) are useful but require infrastructure (localnet docker, funded testnet wallet, Walrus testnet) — run them when relevant; CI gates them separately. See `.github/workflows/` for what CI runs and when.

## Testing instructions

- **Unit tests** are network-free and should pass on every change. Add unit tests for new pure logic.
- **Integration tests** (`ts-sdks/packages/sui-stack-messaging/test/integration/localnet/`) run against a localnet docker stack via testcontainers. They double as living usage examples — read them when extending. See the `develop-on-sui-stack-messaging` skill for the file index.
- **E2E tests** (`test/e2e/`) build the relayer + indexer Docker images via testcontainers and run against real Sui testnet. Require `TEST_WALLET_PRIVATE_KEY`. See `test/e2e/setup-testnet.ts` and the `spin-up-e2e-stack` skill for the orchestration shape.
- **Move tests** live next to sources (`move/packages/*/tests/`). Use `sui move test --path` per package.
- When fixing a bug, add a failing test first and verify the fix flips it.

## Security considerations

- **Encryption invariants are load-bearing.** The envelope encryption format and Seal identity-bytes layout (`[group_id (32)][key_version (8 LE u64)]`) are shared across SDK + Move + relayer. Don't redefine them in a fork. See `docs/sui-stack-messaging/Encryption.md`.
- **Sender verification is independent of the relayer.** Every group member can independently verify a message's sender by re-running `verifyMessageSender` (`ts-sdks/packages/sui-stack-messaging/src/verification.ts`). Changing the canonical-message format means all canonical-SDK clients in the same group stop being able to verify your relayer's clients. See `docs/sui-stack-messaging/Security.md`.
- **The relayer never sees plaintext.** If you fork it, preserve this invariant — don't add endpoints that take or return plaintext message content.
- **Don't commit secrets.** `.env` files are ignored; `.env.example` is the only checked-in form. Never commit a `TEST_WALLET_PRIVATE_KEY`, `ADMIN_SECRET_KEY`, or sponsor-key value.
- **Don't disable signing or hooks** (`--no-verify`, `--no-gpg-sign`, etc.) when committing. If a hook fails, fix the underlying issue.
- **Don't bump dependency versions to bypass `minimumReleaseAge: 2880`** in `ts-sdks/pnpm-workspace.yaml` (a 2-day floor on new dependency releases, designed to reduce supply-chain attack surface). Mysten packages are excluded.

## Commit and PR conventions

- **Conventional Commits** style (`feat:`, `fix:`, `docs:`, `chore:`, `refactor:` …) is used in recent history (see `git log`). Match the existing style.
- **Changesets** govern SDK version bumps. If your change affects the public API of any TS package under `ts-sdks/packages/`, run `pnpm changeset` and commit the generated `.changeset/*.md` along with your code change. See `ts-sdks/RELEASING.md`.
- **One logical change per PR.** Don't bundle a refactor with a feature.
- **Don't push to `main` directly.** Open a PR. Don't force-push to shared branches.
- **Don't amend or rewrite commits** that have already been pushed and reviewed unless explicitly asked.

## Working pattern recommendations

- For multi-step tasks, prefer parallel exploration / read-only investigation before editing. The repo is large; `find`, `grep`, and the `docs/sui-stack-messaging/` index get you oriented fastest.
- Verify any command in this file against `package.json` / `Cargo.toml` / `Move.toml` before reporting "done" — those are the source of truth.
- For Move language fundamentals, defer to general Sui Move documentation; this file only covers what's specific to this repo.
