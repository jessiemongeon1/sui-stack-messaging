# AGENTS.md — relayer/

This is the **reference** Rust axum relayer for Sui Stack Messaging. It is designed to be forked and extended (custom storage backend, auth middleware, sponsor-key strategy, observability, Nautilus runtime, etc.). The repo-root `AGENTS.md` still applies; this file adds relayer-specific commands and invariants.

## Repo layout (this subtree)

```
relayer/
  src/                          axum handlers, auth pipeline, storage, sync
    auth/                         signature, membership, permissions, middleware
    handlers/                     health, messages
    models/                       message, attachment, membership
    services/                     event_parser, membership_sync, walrus_sync
    storage/                      adapter + memory implementation
    walrus/                       Walrus client + types
  tests/                        integration tests (some `--ignored` hit Walrus testnet)
  diagrams/                     architecture diagrams
  docs/                         relayer-specific docs (Postman collection, etc.)
  Cargo.toml / Cargo.lock       Rust deps
  rust-toolchain.toml           pinned: stable + clippy + rustfmt
  Dockerfile                    container image
  docker-compose.yml            local stack
  .env.example                  required env (GROUPS_PACKAGE_ID at minimum)
  README.md                     full reference (auth, API, storage, sync)
```

## Commands

```bash
cp .env.example .env                                # then set GROUPS_PACKAGE_ID
cargo run                                            # dev server on :3000
cargo test                                           # network-free tests
cargo test -- --ignored                              # tests that hit Walrus testnet
cargo fmt
cargo clippy --all-targets -- -D warnings           # MUST pass before declaring done

docker compose up                                    # containerized alternative
```

## Required env (.env)

- `GROUPS_PACKAGE_ID` — testnet `0xba8a26d42bc8b5e5caf4dac2a0f7544128d5dd9b4614af88eec1311ade11de79` or mainnet `0x541840ae7df705d1c6329c22415ed61f9140a18b79b13c1c9dc7415b115c1ba8` (these are sui-groups package IDs, not sui-stack-messaging).
- Other variables in `.env.example`: `SUI_RPC_URL`, `PORT`, `REQUEST_TTL_SECONDS`, `STORAGE_TYPE`, `MEMBERSHIP_STORE_TYPE`, Walrus publisher/aggregator URLs and sync tuning, `RUST_LOG`.

## Toolchain

- Rust: stable (managed by `rust-toolchain.toml`).
- Components: `clippy` + `rustfmt` (both required).

## Hard invariants

- **Do not break the wire protocol** that the SDK (`RelayerTransport`) and the `walrus-discovery-indexer` consume. If you must change types in `src/models/` or `src/walrus/types.rs` that round-trip JSON to the SDK or are persisted to Walrus, update all three sides + `../docs/sui-stack-messaging/Relayer.md` together. (Also enforced as a path-scoped rule under `.claude/rules/`.)
- **The relayer never sees plaintext.** Forks must preserve this invariant — do not add endpoints that take or return plaintext message content. The relayer's job is sponsor + persist; encryption lives in the SDK.
- **Preserve the auth pipeline shape** when forking — see `README.md` and `src/auth/README.md` for the canonical request → signature-verify → membership-check → permission-check flow. Custom middleware should plug in, not replace.
- **Never commit `.env`** — only `.env.example`. Do not commit sponsor keys, admin secret keys, or test wallet private keys.
- **`cargo clippy --all-targets -- -D warnings` must pass** — enforced in CI; do not merge with warnings.

## When to fork vs implement a custom `RelayerTransport`

- Fork **this Rust code** if you need: custom storage backend (PostgreSQL, S3), Nautilus-style enclave runtime, native integration with existing Rust infra.
- Implement a custom **TS `RelayerTransport`** (in your SDK consumer app) if you need: BYO sponsor, lightweight serverless endpoint, no Rust toolchain. See the `configure-custom-relayer-transport` skill.

## Documentation pointers

- `README.md` (this directory) — full reference: auth pipeline, API, storage, sync.
- `src/auth/README.md` — auth pipeline internals.
- `docs/messaging-relayer.postman_collection.json` — API examples.
- `diagrams/` — architecture diagrams.
- `../docs/sui-stack-messaging/Relayer.md` — protocol spec the relayer implements (required reading before any wire-protocol change).

## Skills

- `.claude/skills/spin-up-relayer/SKILL.md` — local dev setup.
- `.claude/skills/develop-relayer/SKILL.md` — fork-and-extend recipe.

## Pre-commit checks

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```
