# AGENTS.md — agenttrace-qwen-bridge

> **For:** AI agents (Miko / Hermes / Claude Code / OpenCode sub-agents) starting a session in this repo.
> **Read this first.** If you are a human, see [`README.md`](README.md) for project context.

## What this project is

This is **Kolya's third repo** in the agenttrace stack. It bridges Qwen Code and GigaCode (community agent CLIs that are forks of Gemini CLI / Qwen Code) into the local **agenttrace** daemon via their HTTP-hook extension API.

Two upstream sibling repos in this stack:

| Repo | Role |
|---|---|
| `nikolay-grudanov/agenttrace` (fork of `raindrop-ai/workshop`) | Daemon on `:5899` |
| `nikolay-grudanov/agenttrace-opencode-plugin` (fork of `@raindrop-ai/opencode-plugin`) | Plugin hooks OpenCode SDK events |
| **`nikolay-grudanov/agenttrace-qwen-bridge` (this repo)** | **Bridge HTTP-hook events from Qwen Code / GigaCode to the daemon** |

The umbrella directory is `~/workspase/projects/opencode-workshop-stack/`. Read its `AGENTS.md` first to understand hard rules.

## Stack

- **Language:** TypeScript (strict, ES2022).
- **Runtime:** Node.js ≥ 18 (every Qwen Code / GigaCode user has Node preinstalled — Qwen Code ships as a Node CLI).
- **HTTP server:** stdlib `node:http` only. No Express, no Fastify. Target: < 100 KB installed.
- **Validation:** `zod` (one dependency).
- **Build:** `tsc` to `dist/`.
- **Test:** `node --test` (built-in, no test framework dependency).

## What lives here

- `src/` — HTTP server, hook handlers, translator, shipper, config, logger.
- `tests/` — `node --test` unit + integration tests.
- `scripts/` — install/smoke helpers.
- `ai-docs/PLAN.md` — feature tracker (F-NNN).
- `ai-docs/specs/` — wire contract, design notes.
- `dist/` — built output (gitignored).

## Public package

- **npm:** `@grudanov-nikolay/agenttrace-qwen-bridge`
- **GitHub:** `nikolay-grudanov/agenttrace-qwen-bridge`
- **License:** MIT, dual copyright (Raindrop AI + Nikolai Grudanov).

## Build & test (planned)

```bash
bun install
bun x tsc --noEmit      # typecheck
bun test                # node --test tests/
node scripts/smoke-test.sh
```

## What an agent must read before acting

1. **This file** (you are here).
2. **`ai-docs/PLAN.md`** — current plan, F-NNN features with `- [ ]` / `- [x]` todos.
3. **Umbrella `~/workspase/projects/opencode-workshop-stack/AGENTS.md`** — hard rules (no auto-commit/push, no daemon restart, etc.).
4. (When in doubt) **`openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`** in the umbrella — full design rationale and stage plan.

## Hard rules (Kolya enforcement)

These are inherited from the umbrella `AGENTS.md` and apply **identically** to this repo:

1. **No auto-commit, no auto-push.** `git commit` requires Kolya's explicit «приступай», «согласен», «коммить» etc. `git push` requires explicit Kolya command — never auto.
2. **No daemon restart by the assistant.** Bridge does not need daemon restart on its own (agenttrace daemon restart is Kolya's action), but if you change agenttrace daemon code in the same F-NNN flow, **propose restart**, do not execute.
3. **No edits to `~/.hermes/config.yaml` or `~/.hermes/.env`.** Hermes configuration is Kolya-managed.
4. **`ai-docs/PLAN.md` is the single source of truth.** Update in the same commit as the code change. New Features at the top (newest F-number first). Closed Features move to bottom with `Closed YYYY-MM-DD`.
5. **Bridge is read-only toward Workshop daemon.** We never inject `additionalContext`, never block tool calls. Per F-023 review Q9.
6. **Bridge binds to `127.0.0.1` only.** Local-only debugger. No LAN exposure. No cloud.
7. **Wire contract is canonical.** Any change to the JSON shape the bridge emits to `:5899/v1/` must update `ai-docs/specs/F-023-wire-contract.md` in the same commit. The corporate GigaCode-adapter author reads this document.

## Working conventions

- **One commit per logical change.** Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`.
- **Reference F-NNN in commit body** (e.g. `Refs F-023.` or `Closes F-023-M1`).
- **Tag convention:** `v<major>.<minor>.<patch>` (no `-kolya.<N>` suffix — that was an in-house dev channel pre-1.0; now we publish under semver).
- **Git workflow:** commit locally, wait for Kolya's «пуш» before pushing.

## Do NOT touch

- `node_modules/`, `dist/` — gitignored.
- `~/.hermes/` — Kolya-managed.
- Any file under the umbrella `opencode-workshop/` or `opencode-workshop-plugin/` directories — they are sibling repos with their own history.

## Quick reference

| Need | Path |
|---|---|
| Plan / todos | `ai-docs/PLAN.md` |
| Wire contract (for GigaCode adapter author) | `ai-docs/specs/F-023-wire-contract.md` (planned, Stage 1) |
| Bridge config | `src/config.ts` (planned) |
| HTTP server entry | `src/server.ts` (planned) |
| Umbrella AGENTS.md | `~/workspase/projects/opencode-workshop-stack/AGENTS.md` |
| agenttrace daemon AGENTS.md | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/AGENTS.md` |
| Stack status | `~/workspase/projects/opencode-workshop-stack/STATUS.md` |

---

*Maintained by Nikolai Grudanov (MIFI, M.Sc. 01.04.02) via Miko (Hermes Agent). Scaffolded 2026-09-17; renamed to agenttrace on 2026-09-17 (F-024).*
