# CLAUDE.md

@AGENTS.md

## Claude Code specifics

- **Per-subproject guidance auto-loads.** When you `cd` into `move/`, `ts-sdks/`, `relayer/`, `walrus-discovery-indexer/`, `chat-app/`, or `publish/`, the subproject's `CLAUDE.md` is loaded automatically and pulls in its `AGENTS.md`. Don't re-derive subproject conventions — read what's already loaded.
- **Path-scoped rules under `.claude/rules/`** trigger when you read matching files: `no-edit-generated-codegen.md` fires inside `ts-sdks/packages/sui-stack-messaging/src/contracts/`; `wire-protocol-cross-impact.md` fires when you touch protocol-shared files.
- **Skills are listed in `AGENTS.md` above** — invoke with `/<skill-name>`. Each `SKILL.md` is a focused, single-task runbook.
