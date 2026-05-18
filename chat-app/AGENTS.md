# AGENTS.md — chat-app

This is the **reference** Vite + React 19 demo app for `sui-stack-messaging`, intended to be forked as a starting point for a production UI. It is *not* a production-hardened chat app. Root-level guidance in [`../AGENTS.md`](../AGENTS.md) still applies; this file adds chat-app-specific commands and invariants.

## Repo layout (this subtree)

```
chat-app/
  src/                          React app source
  docs/                         system design doc
  index.html                    Vite entry
  package.json                  scripts + locally-linked SDK
  pnpm-workspace.yaml           (own workspace, not part of ts-sdks/)
  vite.config.ts                Vite config
  .env.example                  required env
  README.md                     ~190 lines: setup, system overview
```

## Commands

```bash
pnpm install
pnpm build:deps                # builds the linked SDK from ../ts-sdks/
pnpm dev                       # vite :5173
pnpm build                     # runs build:deps + tsc -b + vite build
pnpm preview
```

## The locally-linked SDK

`package.json` declares: `"@mysten/sui-stack-messaging": "link:../ts-sdks/packages/sui-stack-messaging"`.

This means:

- Changes to the SDK in `../ts-sdks/packages/sui-stack-messaging/src/` are picked up after re-running **`pnpm build:deps`** from this directory (which builds the linked SDK so its `dist/` reflects current source).
- A clean `pnpm install` here will fail if `../ts-sdks/` hasn't been installed first.
- `pnpm build` already chains `build:deps`; the manual step matters mostly during `pnpm dev`.
- This is fine for local dev. A fork that goes to production should change this dependency to a pinned `@mysten/sui-stack-messaging` version from npm.

## Toolchain

- pnpm (matches root constraint of `>=10.17.0`)
- React 19
- Vite 6 (no webpack/CRA)
- TypeScript via `tsc -b` (project references)
- Tailwind v4 (`@tailwindcss/vite`)

## Hard invariants

- **This is a reference / demo app, not production code.** Don't treat its choices (auth flow, error handling, UX patterns) as canonical. Fork it as a *starting point* and replace whatever doesn't fit your needs.
- **After any SDK change, run `pnpm build:deps`** — otherwise this app picks up stale SDK output and you get confusing behavior.
- **`.env` is gitignored; `.env.example` is the only checked-in form.** Never commit a private key, RPC token, or other secret. All app env vars are `VITE_*` prefixed and bundled into the client — treat them as public.
- **Don't introduce production hardening** (rate limits, multi-region failover, complex error UI) into this reference app. If you want to ship that, fork the repo into your own project.
- **No test runner is wired up here.** Don't add one in passing; if tests are needed, propose the choice first.

## Documentation pointers

- `README.md` (this directory) — setup + feature overview.
- `docs/SYSTEM_DESIGN.md` (this directory) — architecture deep-dive (read before making structural changes).
- `../docs/sui-stack-messaging/Setup.md` and `../docs/sui-stack-messaging/Examples.md` — generic SDK setup/usage that this app demonstrates.

## Skills

- `.claude/skills/spin-up-e2e-stack/SKILL.md` — full local stack with this app on top.
- `.claude/skills/integrate-sui-stack-messaging/SKILL.md` (builder-facing) — describes the SDK integration patterns this app exemplifies.

## Pre-commit checks

```bash
pnpm build                     # build:deps + tsc -b + vite build (catches type errors)
```
