# AGENTS.md — agenttrace-qwen-bridge

> **For:** AI agents (Miko / Hermes / Claude Code / OpenCode sub-agents) starting a session in this repo.
> **Read this first.** If you are a human, see [`README.md`](README.md) for project context.

## What this project is

This is **Kolya's third repo** in the agenttrace stack. It bridges Qwen Code and GigaCode (community agent CLIs — forks of Gemini CLI / Qwen Code) into the local **agenttrace** daemon by reading their JSONL event streams and impersonating the OpenCode-plugin wire format.

**Architecture (F-025, flipped 2026-09-18 — supersedes the original F-023 HTTP-hook design):** Qwen Code runs only in its interactive TUI (Kolya's rule, D8). The TUI's dual-output mode (`--json-file <fifo>`) streams full-fidelity JSONL events while the TUI renders on stdout. A `SessionStart` hook auto-spawns `bridge watch`, which reads `session_id` + `cwd` from the hook's stdin JSON, creates `/tmp/agenttrace-qwen/$SESSION_ID.fifo`, and tails it. **No HTTP server** (D1/D9 — deferred indefinitely).

Two upstream sibling repos in this stack:

| Repo | Role |
|---|---|
| `nikolay-grudanov/agenttrace` (fork of `raindrop-ai/workshop`) | Daemon on `:5899` |
| `nikolay-grudanov/agenttrace-opencode-plugin` (fork of `@raindrop-ai/opencode-plugin`) | Plugin hooks OpenCode SDK events |
| **`nikolay-grudanov/agenttrace-qwen-bridge` (this repo)** | **Bridge JSONL events from Qwen Code / GigaCode TUI to the daemon** |

The umbrella directory is `~/workspase/projects/agenttrace-stack/`. Read its `AGENTS.md` first to understand hard rules.

## Stack

- **Language:** TypeScript (strict, ES2022).
- **Runtime:** Node.js ≥ 18 (every Qwen Code / GigaCode user has Node preinstalled — Qwen Code ships as a Node CLI).
- **Ingest:** stdlib `node:readline` + `node:fs` only — FIFO/file tail, no HTTP server (F-025 D1).
- **Validation:** `zod` (one dependency). Single schema for both Qwen and GigaCode shapes (D7 — verified 2026-09-22 against a 5403-line GigaCode fixture; both emit the `{"type":"system","subtype":...}` envelope).
- **Build:** `tsc` to `dist/`.
- **Test:** `node --test` (built-in, no test framework dependency).

## What lives here

- `src/` — CLI (`watch` / `tail` / `install` / `uninstall`), session-watch hook entrypoint, FIFO tail reader, zod wire-format schemas, translator, shipper, config, logger.
- `tests/` — `node --test` unit + integration tests (fixtures under `tests/fixtures/`).
- `scripts/` — install/smoke helpers.
- `ai-docs/PLAN.md` — feature tracker (F-NNN).
- `ai-docs/specs/` — wire contract, design notes.
- `dist/` — built output (gitignored).

## Public package

- **npm:** `@grudanov-nikolay/agenttrace-qwen-bridge` (not yet published — no code yet)
- **GitHub:** `nikolay-grudanov/agenttrace-qwen-bridge` (not yet pushed — awaits Kolya's `gh repo create`)
- **License:** MIT, dual copyright (Raindrop AI + Nikolai Grudanov).

## Build & test (planned)

```bash
bun install
bun x tsc --noEmit      # typecheck
bun test                # node --test tests/
```

## What an agent must read before acting

1. **This file** (you are here).
2. **`ai-docs/PLAN.md`** — current plan, F-NNN features with `- [ ]` / `- [x]` todos. F-025 holds all architecture decisions (D1–D11) and the fixture requirements.
3. **Umbrella `~/workspase/projects/agenttrace-stack/AGENTS.md`** — hard rules (no auto-commit/push, no daemon restart, etc.).
4. (When in doubt) **`openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`** in the daemon repo — original design rationale (pre-flip; F-025 PLAN entry records what changed).

## Hard rules (Kolya enforcement)

These are inherited from the umbrella `AGENTS.md` and apply **identically** to this repo:

1. **No auto-commit, no auto-push.** `git commit` requires Kolya's explicit «приступай», «согласен», «коммить» etc. `git push` requires explicit Kolya command — never auto.
2. **No daemon restart by the assistant.** Bridge does not need daemon restart on its own (agenttrace daemon restart is Kolya's action), but if you change agenttrace daemon code in the same F-NNN flow, **propose restart**, do not execute.
3. **No edits to `~/.hermes/config.yaml` or `~/.hermes/.env`.** Hermes configuration is Kolya-managed.
4. **`ai-docs/PLAN.md` is the single source of truth.** Update in the same commit as the code change. New Features at the top (newest F-number first). Closed Features move to bottom with `Closed YYYY-MM-DD`.
5. **Bridge is read-only toward the agenttrace daemon.** We never inject `additionalContext`, never block tool calls. Per F-023 review Q9.
6. **Bridge talks to `127.0.0.1` only.** Local-only debugger. No LAN exposure. No cloud.
7. **Wire contract is canonical.** Any change to the JSON shape the bridge emits to `:5899/v1/` must update `ai-docs/specs/F-025-wire-contract.md` in the same commit. The corporate GigaCode-adapter author reads this document.
8. **No code before fixtures.** F-023/F-025 implementation is blocked on Kolya's real `qwen.jsonl` TUI fixture (+ sub-agent file-layout verification). Zod schemas are written against real traces, not guesses.

## Working conventions

- **One commit per logical change.** Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`.
- **Reference F-NNN in commit body** (e.g. `Refs F-025.` or `Closes F-025-M1`).
- **Tag convention:** `v<major>.<minor>.<patch>` (no `-kolya.<N>` suffix — that was an in-house dev channel pre-1.0; now we publish under semver).
- **Git workflow:** commit locally, wait for Kolya's «пуш» before pushing.

## Do NOT touch

- `node_modules/`, `dist/` — gitignored.
- `~/.hermes/` — Kolya-managed.
- Any file under the umbrella `agenttrace/` or `agenttrace-opencode-plugin/` directories — they are sibling repos with their own history.

## Quick reference

| Need | Path |
|---|---|
| Plan / todos | `ai-docs/PLAN.md` |
| Wire contract (for GigaCode adapter author) | `ai-docs/specs/F-025-wire-contract.md` (planned — written with the first implementation commit) |
| Bridge config | `src/config.ts` (planned) |
| CLI entrypoint | `src/cli.ts` (planned — `watch` / `tail` / `install` / `uninstall`) |
| FIFO tail reader | `src/tail.ts` (planned) |
| Umbrella AGENTS.md | `~/workspase/projects/agenttrace-stack/AGENTS.md` |
| agenttrace daemon AGENTS.md | `~/workspase/projects/agenttrace-stack/agenttrace/AGENTS.md` |
| Stack status | `~/workspase/projects/agenttrace-stack/STATUS.md` |

---

*Maintained by Nikolai Grudanov (MIFI, M.Sc. 01.04.02) via Miko (Hermes Agent). Scaffolded 2026-09-17; renamed to agenttrace on 2026-09-17 (F-024); architecture flipped to JSON-stream 2026-09-18 (F-025).*
