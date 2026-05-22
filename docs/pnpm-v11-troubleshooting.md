# pnpm v11 — build scripts and `overrides` (`ERR_PNPM_IGNORED_BUILDS` / `ERR_PNPM_LOCKFILE_CONFIG_MISMATCH`)

A fresh `pnpm install` (or `docker build`) for the reference TypeScript projects — `walrus-discovery-indexer/` and `chat-app/` — can fail on **pnpm v11** with one or both of:

```
[ERR_PNPM_IGNORED_BUILDS] Ignored build scripts: esbuild
[ERR_PNPM_LOCKFILE_CONFIG_MISMATCH] Cannot proceed with the frozen installation.
The current "overrides" configuration doesn't match the value found in the lockfile
```

The Rust relayer is unaffected (no pnpm).

## Root cause — one cause, two symptoms

This repo configures pnpm through the **`pnpm` field in `package.json`** (build-script approvals **and** dependency `overrides`). **pnpm v11 stopped reading that field** — those settings moved to `pnpm-workspace.yaml`. pnpm **v10** tolerated the old layout (warnings, exit 0); pnpm **v11** turns it into hard errors (exit 1), so a setup that worked before now fails on a fresh machine.

- **`ERR_PNPM_IGNORED_BUILDS: esbuild`** — esbuild ships a `postinstall`, and pnpm v11 refuses to run unapproved build scripts. esbuild's native binary actually arrives via the `@esbuild/<platform>` optional dependency, so the postinstall is safe to skip.
- **`ERR_PNPM_LOCKFILE_CONFIG_MISMATCH` ("overrides")** — the committed lockfile records `overrides` (security version pins) that v11 no longer reads from `package.json`, so `--frozen-lockfile` rejects the install.

`--frozen-lockfile` does **not** downgrade either error.

## Authoritative context

- **pnpm v10 → v11 migration guide: https://pnpm.io/migration** — the field move, the new `allowBuilds` map that replaces `onlyBuiltDependencies` / `ignoredBuiltDependencies` / `neverBuiltDependencies`, where `overrides` now lives, and the automated `pnpm codemod run pnpm-v10-to-v11`.
- pnpm supply-chain security: https://pnpm.io/supply-chain-security

## Remedies

The repo installs, builds, and runs cleanly on **pnpm 10.x** (its declared floor, `pnpm >=10.17.0`); the breakage only appears on pnpm 11.

### Docker

Pin pnpm 10 **in the image** — change `corepack prepare pnpm@latest` to `corepack prepare pnpm@10` in the Dockerfile. Contained to the build, no host impact, and the `overrides` security pins are preserved. Builds green with no other edits.

### Host

Use the developer's **existing** pnpm; **do not globally switch it.** `corepack prepare … --activate` changes the default pnpm for *all* of the user's projects, so an agent must never run it as a fix.

- On pnpm v11, `pnpm install --ignore-scripts` clears the esbuild gate. A normal (non-`--frozen`) host install never hits the `overrides` mismatch — it just refreshes the local lockfile.
- To pin pnpm, do it **per-project** with a `packageManager` field — never a global activate.
- Caveat: `pnpm <script>` (e.g. `pnpm dev`) re-checks deps first and can re-trigger the build gate even after a clean `--ignore-scripts` install. If so, run the binary directly (`./node_modules/.bin/tsx watch src/index.ts`, `./node_modules/.bin/vite`) or the compiled output (`pnpm build` then `node dist/index.js`).

### Canonical `ts-sdks/`

Do **not** edit `ts-sdks/` (the published canonical SDK). If you need it built locally — only `chat-app` Mode B / SDK contributors — use a project-scoped pnpm 10 (e.g. `npx pnpm@10 …`), or simply depend on the published `@mysten/sui-stack-messaging` package instead of the workspace `link:`.

## Proper repo-side fix (maintainers)

Follow the migration guide: move the whole `package.json` `pnpm` field into each affected project's `pnpm-workspace.yaml` (`overrides:` plus `allowBuilds: { esbuild: false }`, optionally `ignoredBuiltDependencies: [esbuild]` for v10 too), and have each Dockerfile `COPY pnpm-workspace.yaml` before `pnpm install`. Not applied today. Do not edit canonical `ts-sdks/` outside the changeset/release process.
