# PLAN.md — agenttrace-qwen-bridge (Kolya's bridge repo, formerly `openworkshop-qwen-bridge`)

> **Single source of truth for all development work in this repo.**
> Features at the top (newest first), each with checkboxes. Update in the same commit as the code change.

## Conventions

- **Feature = a vertical slice of work** (one user-visible capability, one bug fix, or one cleanup).
- **Todo = a single atomic step** inside a Feature. Marked `- [ ]` (pending) or `- [x]` (done).
- **F-NNN = Feature ID**, assigned in order of creation. Never reused.
- **Order:** open Feature at the top of the file. Newest F-number first.
- **Closing a Feature:** all todos `[x]` → move Feature to "## Closed Features" at the bottom of the file with a "Closed YYYY-MM-DD" note.

## Roadmap (Tier 1, next-up) — 2026-09-17

First public alpha target: bridge runs locally, ingests Qwen Code hook events into agenttrace daemon, ships tagged spans with `agent_provider="qwen_code"`. GigaCode support via wire contract (corporate agent implements GigaCode-specific handler in their own time).

- **F-023. Bridge scaffold + wire contract** — repo bootstrap (this stage), wire-contract spec, zod schemas, translator skeleton, smoke test recipe. See umbrella `~/workspase/projects/opencode-workshop-stack/opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md` for the full cross-repo design.

## Active Features

### F-024 — Rename stack: `openworkshop-qwen-bridge` → `agenttrace-qwen-bridge`

**Context:** Companion rename to the umbrella stack rename (F-024 in `opencode-workshop` and `opencode-workshop-plugin`). As of 2026-09-17 this bridge repo is renamed from `openworkshop-qwen-bridge` → `agenttrace-qwen-bridge` to align with the agenttrace identity.

**Files to edit:**
- `package.json` — `name`, `bin`, `description`, `homepage`, `bugs.url`, `repository.url`, `keywords`
- `README.md` — title, install command, description, public package links
- `AGENTS.md` — title, sibling-repo table, public package block, hard-rules wording, Quick Reference
- `ai-docs/PLAN.md` — this entry (also: `## Roadmap` description, F-023 description, `git init + first commit` todo marked done)

**Out of scope:**
- `ai-docs/specs/handoff/F-023-corporate-handshake.md` — historical handoff to the corporate GigaCode author dated 2026-09-17. References to `openworkshop-qwen-bridge` paths are part of the historical record and stay intact.
- New `git init` (this repo never had a git init before this rename — `git init` is the next todo).
- Remote push (`git remote add origin` + `git push`) — Kolya action.

**Deferred to separate Kolya-authorized steps:**
- `npm publish @grudanov-nikolay/agenttrace-qwen-bridge@<future>`
- `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` (first push to a brand-new remote)

**Todos:**
- [x] `package.json` — name, bin, description, homepage, bugs.url, repository.url, keywords
- [x] `README.md` — header, install command, public package links, install/recipe
- [x] `AGENTS.md` — title, sibling-repo table, public package block, hard-rules wording, Quick Reference, footer
- [x] `ai-docs/PLAN.md` — title + Roadmap + F-023 description + F-024 entry
- [ ] `git init` + first commit (next step — was a deferred todo under F-023)
- [ ] `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` + first push (Kolya action)

### F-023 — Bridge for Qwen Code + GigaCode → agenttrace daemon

**Context:** see umbrella `openspec/changes/archive/F-023-multi-source-ingestion/proposal.md` for full design. This PLAN.md entry is a **mirror** for repo-local todos — canonical decisions live in the umbrella proposal.

**Repo-local scope:**
- `src/server.ts` — stdlib HTTP server on `127.0.0.1:5898`, routes `/qwen/hook`, `/gigacode/hook`, `/health`, `/ready`.
- `src/handler-qwen.ts` — Qwen Code `PreToolUse`/`PostToolUse`/`Stop`/`SubagentStop` → translator.
- `src/handler-gigacode.ts` — placeholder stub for the wire contract (corporate agent fills in real handler against `ai-docs/specs/F-023-wire-contract.md`).
- `src/translator.ts` — pure function, hook event → Workshop-span shape.
- `src/wire-format.ts` — zod schemas (Qwen input + Workshop output).
- `src/shipper.ts` — POSTs translated spans to `localhost:5899/v1/` (re-uses OpenCode-plugin wire format).
- `src/config.ts` — env-var loader with `isLocalUrl()` guard (F-005 pattern from plugin).
- `src/logger.ts` — leveled logger.
- `tests/handler-qwen.test.ts` — 6+ unit tests on translated output.
- `tests/wire-format.test.ts` — round-trip zod tests.
- `tests/shipper.test.ts` — HTTP mocking.
- `ai-docs/specs/F-023-wire-contract.md` — canonical document for the corporate GigaCode author.
- `.github/workflows/ci.yml` — `tsc --noEmit` + `node --test`.

**Cross-repo impact:**
- **No changes** to `agenttrace-opencode-plugin` (the bridge impersonates its wire format).
- **Minimal changes** to `agenttrace`: read `agent_provider` field, UI source badge + facet (handled by umbrella workshop's F-023 entry).

**Todos (Stage 0 only — repo bootstrap, in progress 2026-09-17):**
- [x] Create repo at `~/workspase/projects/agenttrace-qwen-bridge/` (originally scaffolded as `openworkshop-qwen-bridge`, renamed 2026-09-17 F-024)
- [x] Write `package.json` (npm namespace `@grudanov-nikolay/agenttrace-qwen-bridge`)
- [x] Write `tsconfig.json` (strict, ES2022)
- [x] Write `.gitignore` (node_modules/, dist/, .zcode/, .env)
- [x] Write `LICENSE` (dual copyright Raindrop AI + Nikolai Grudanov)
- [x] Write `README.md` (minimal pointer)
- [x] Write `AGENTS.md` (within-repo)
- [x] Write `ai-docs/PLAN.md` (this file)
- [ ] Update umbrella `~/workspase/projects/opencode-workshop-stack/AGENTS.md` to reference this repo
- [ ] `git init` + first commit (requires Kolya's «приступай»)
- [ ] Publish to npm `0.0.1` (after M1.5 smoke test, requires Kolya's `npm publish`)

**Future milestones (Stages 1-3 of umbrella proposal):**
- Stage 1 (M1.1–M1.3): Wire contract + zod schemas + round-trip tests.
- Stage 2 (M2.1–M2.3): Translator + shipper + HTTP server.
- Stage 3 (M3.1–M3.5): Hardening + CI + first npm release.

Each stage = one Feature-step, each commit independently buildable/testable.

## Closed Features

_(none yet — this repo is brand new)_

---

*Maintained by Miko (Hermes Agent) under Kolya's direction. Update this file when the repo state changes meaningfully.*
