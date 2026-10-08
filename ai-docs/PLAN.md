# PLAN.md — agenttrace-qwen-bridge (Kolya's bridge repo)

> **Single source of truth for all development work in this repo.**
> Features at the top (newest first), each with checkboxes. Update in the same commit as the code change.

## Conventions

- **Feature = a vertical slice of work** (one user-visible capability, one bug fix, or one cleanup).
- **Todo = a single atomic step** inside a Feature. Marked `- [ ]` (pending) or `- [x]` (done).
- **F-NNN = Feature ID**, assigned in order of creation. Never reused.
- **Order:** open Feature at the top of the file. Newest F-number first.
- **Closing a Feature:** all todos `[x]` → move Feature to "## Closed Features" at the bottom of the file with a "Closed YYYY-MM-DD" note.

## Roadmap (Tier 1, next-up) — 2026-09-18

Bridge runs locally under Qwen Code's interactive TUI. Each session starts → `SessionStart` hook spawns `bridge watch` → bridge reads JSONL events from a FIFO (`--json-file <fifo>` writes there in real time while the TUI renders on stdout) → translator converts to Workshop-spans → shipper POSTs to `localhost:5899/v1/` with `agent_provider="qwen_code"`.

- **F-025 (this). Architecture flip — JSON-stream primary, SessionStart-hook wrapper.** See "Decisions" below.
- **F-023.** Wire contract + zod schemas + translator + shipper + `bridge watch`/`bridge tail` + `bridge install` settings.json patcher. Implementation blocked on Kolya sending real gigacode.jsonl + qwen.jsonl fixture files (with sub-agent + error cases).
- **F-024 (closed-pending).** Stack rename (`openworkshop-qwen-bridge` → `agenttrace-qwen-bridge`). Git init + first commit still pending Kolya's «приступай».

## Active Features

### F-025 — Architecture: JSON-stream bridge with SessionStart-hook auto-attach

**Context:** 2026-09-18 architecture session with Kolya. Originally F-023 was designed as an HTTP-hook bridge (`:5898` receiving `PreToolUse`/`PostToolUse`/`Stop`/`SubagentStop` from Qwen Code's hook system). Smoke-testing on Kolya's machine revealed two problems:

1. **Qwen Code is run only in interactive TUI mode** — headless `-y -o stream-json` was a wrong assumption. The native dual-output mode (`--json-file` / `--json-fd`) IS the right mechanism for TUI: TUI renders on stdout, JSON events stream to the file/fifo/fd in parallel.
2. **HTTP hooks require `settings.json` editing + `allowPrivateNetworkHooks` opt-in** (per `https://qwenlm.github.io/qwen-code-docs/en/users/features/hooks/`). Setup friction.

Kolya proposed: **let `SessionStart` hook auto-attach the bridge** so there's zero per-session setup. Decision: dual-mode CLI (`bridge watch` for SessionStart-hook auto-attach + `bridge tail` for manual invocation), JSON-stream is the only ingest path (HTTP-bridge deferred indefinitely — only the corporate GigaCode fork without TUI would need it).

**Awaiting fixtures from Kolya (BLOCKER — no code before this):**

The whole bridge implementation depends on knowing the **exact** JSONL shapes Qwen Code 0.23.4 and the GigaCode fork emit. Two real interactive-TUI session traces are required. Both must be **unredacted** (full tool_input bodies, full tool_result contents, full model reasoning where present). Place them as attachments in this chat or commit them under `tests/fixtures/` once the repo is git-initialised.

**Fixture 1 — `qwen.jsonl`** (interactive TUI session, captured via `--json-file /tmp/x.jsonl`):
- Must contain `system/init` (or `session_start`) event with full `cwd`, `tools[]`, `mcp_servers`, `model`, `qwen_code_version`, `agents[]`.
- Must contain ≥ 1 assistant turn with `thinking` content blocks (so we can model reasoning spans correctly).
- Must contain ≥ 1 `tool_use` event whose `input` is non-trivial (≥ 1 KB), and the matching `tool_result` (success or error).
- Must contain ≥ 1 sub-agent invocation (`tool_use.name == "agent"` or `task` or similar) — full input prompt + the full child session's events. This is the **only** way to validate `parent_tool_use_id` nesting for sub-agent spans.
- Must contain ≥ 1 tool error: `tool_result.is_error == true` or `result.is_error == true`, to validate F-013 `status=ERROR` mapping in Workshop.
- Ideally 5–20 minutes of real work, not a smoke test — long enough that we see a full assistant message stream (`stream_event` chunks) interleaved with `assistant` finalised blocks.

**Fixture 2 — `gigacode.jsonl`** (corporate GigaCode fork, same capture method):
- Same content requirements as fixture 1.
- Specifically confirm or correct the field-shape differences already observed in Kolya's screenshot (IMG_0621.jpeg): `type:"session_start"` (vs Qwen `system/init`), `type:"assistant"` with `message.content[].type` set to `thinking` / `text` / `tool_use`, `type:"user"` for `tool_result`, `type:"result"` with `duration_ms`.
- If `parent_tool_use_id` is absent or differently positioned in sub-agent events, document the exact shape so `src/wire-format.ts` zod schema can be written without guessing.
- If `session_id` format differs from Qwen's UUIDv4, document it (GigaCode may use the shorter `25102` form from the screenshot — to be confirmed).

**Why these exact requirements:**
- Without sub-agent trace: translator will work for top-level only and need a rewrite the first time a sub-agent is hit live.
- Without tool error trace: F-013 status mapping ships broken; you'll see all Qwen runs as `status=SUCCESS` even when tools crash.
- Without full assistant turn (`thinking` + multiple `text` + `tool_use` blocks in one event): we'll have to guess span boundaries for reasoning vs prose.
- Without `cwd` / `model` / `agents[]`: `runs.metadata` will be incomplete, UI badges will be missing labels.

**Once fixtures are received:**
1. First commit `tests/fixtures/qwen.jsonl` + `tests/fixtures/gigacode.jsonl` (Kolya redacts any secrets — or confirms full-file is OK to commit).
2. Then walk through each todo below in order, with zod schemas asserted against the fixtures.

**Partial fixture received 2026-09-22 from Kolya** (`~/workspase/projects/private-work-staff/json_ggcode/trace_1.json`, 1.6 MB, 5403 lines, gigacode session `26.9.64`, model `vip/GLM-5.3`, no secrets — Kolya confirmed OK to reference but not yet to commit to `tests/fixtures/`).

Confirmed from `trace_1.json`:
- ✅ Event shapes match the Qwen hypothesis (not the IMG_0621.jpeg screenshot guess): GigaCode **also** uses `{"type":"system","subtype":"session_start",...}` and `{"type":"system","subtype":"session_end",...}`. Single zod schema for both, not separate Qwen/GigaCode schemas.
- ✅ `session_id` = UUIDv4 (`6cc302b9-0dc6-4eb8-97fc-1b42576b57b7`). Top-level field, also duplicated in `data.session_id`.
- ✅ `parent_tool_use_id` present at top level but always `null` in this trace (top-level session, no in-flight sub-agent events).
- ✅ Anthropic-style `stream_event`: `message_start`/`message_stop` (37/37) + `content_block_start`/`content_block_delta`/`content_block_stop` (75/5066/75). Index always 0/1 in this trace.
- ✅ `assistant.message.content[]` blocks: `thinking` (with `thinking_delta` stream), `text`, `tool_use`.
- ✅ `tool_use.input` is streamed via `stream_event` `input_json_delta` chunks then finalised in the `assistant` event — so translator must accumulate deltas to reconstruct full input.
- ✅ `user.message.content[]` for `tool_result`: `{tool_use_id, is_error, content}` (string for permission denial, likely array for bash output).
- ✅ `tool_use.id` format: `call_<32 hex chars>` (Anthropic-style, not UUID).
- ✅ `model` can carry vendor prefix: `vip/GLM-5.3`. Shipper must preserve full string in `runs.metadata`.
- ✅ Tool error case confirmed: `{"type":"user",...,"content":[{"type":"tool_result","tool_use_id":"call_...","is_error":true,"content":"Tool \"run_shell_command\" is denied: the tool's default permission is 'deny'."}]}`. F-013 status mapping should distinguish `permission_denied` from generic `runtime_error`.
- ⚠️ `result` event **absent** in this TUI trace (0 events of `type:"result"`). Session finalisation comes from `system/session_end` instead. Translator must treat `system/session_end` as run-finalize, not `result`. **TO CONFIRM against Qwen 0.23.4** — does Qwen TUI also emit `result` events, or only `system/session_end`?
- ⚠️ Sub-agent invocation seen (`tool_use.name:"agent"` with `subagent_type:"sa-developer-researcher"`, large `prompt`) but **child session events are NOT in this trace** — `session_end` came right after. Cannot validate `parent_tool_use_id` semantics inside the child session. **Need a second gigacode fixture where a sub-agent actually runs and returns.**

**Second fixture received 2026-09-23 from Kolya** (`/home/gna/.hermes/webui/attachments/6d3ea12c9f3e/trace_2.json.txt`, 3.3 MB, 12104 lines, gigacode session `26.9.64`, model `vip/GLM-5.3`, main-session-id `7f7459ef-926a-4400-ac29-0acfe076be7f`, cwd `/home/work/20706056@sigma.sbrf.ru/projects/SA/ibp-digital-architecture`).

New confirmed facts from `trace_2.json.txt`:
- ✅ **`result` event STILL absent** (0/12104 lines). GigaCode TUI never emits `result` — session finalisation is exclusively `system/session_end`. Translator can rely on this. **Qwen 0.23.4 still to confirm** — might be different.
- ✅ **`control_request` / `control_response` present** (2+2 events, lines 9269-9270). Subtype `can_use_tool`, fields `{request_id, request:{subtype, tool_name, tool_use_id, input, permission_suggestions[]}, blocked_path}`. Response `{response:{subtype:"success", request_id, response:{allowed: true|false}}}`. **Translator decision: ignore these for span emission** (they're permission dialogs, not actual work) but log `permission_denied` when `allowed:false`. If a Qwen fixture shows `control_request` followed by tool denial without further events — that's how F-013 `permission_denied` reason gets inferred.
- ⚠️ **Sub-agent events MAY OR MAY NOT interleave into the parent trace.** This is a **hypothesis, not a confirmed fact** — the only direct evidence is one tool_result content string (line 9280) saying `"Background agent launched successfully. output_file: /home/work/.../.gigacode/projects/<cwd-hash>/subagents/<main-session-id>/agent-<subagent_type>-<tool_use_id>.jsonl"`. We also observed: 15 `tool_use.name:"agent"` invocations (line 9267), but **0 events in this trace have non-null `parent_tool_use_id`** and **0 events have a different `session_id`**. This is consistent with sub-agent events living in a separate file, but it could ALSO be explained by: (i) the ping-test sub-agents returning too fast to emit any events between launch and result, or (ii) the parent trace simply not recording child events by design. **Kolya to verify** by running a long-running sub-agent and tailing both `~/.gigacode/projects/*/subagents/<sid>/agent-*.jsonl` AND the main trace simultaneously. Until then, design treats option (b) below as default (parent bridge sees only `tool_use` start + `tool_result` final return for each sub-agent invocation; sub-agent internals not visible).
- ✅ **Multiple sub-agent types confirmed** in this trace: `general-purpose`, `sa-developer-antagonist`, `sa-developer-architect`, `sa-developer-asyncapi-writer`, `sa-developer-code-researcher`, `sa-developer-dbml-writer`, `sa-developer-dictionary-writer`, `sa-developer-mcp-writer`, `sa-developer-openapi-writer`, `sa-developer-researcher`, `sa-developer-reviewer`, `sa-developer-role-model-writer`, `sa-developer-system-func-writer`, `Explore`, `statusline-setup`. So `subagent_type` is free-form string, not enum.
- ✅ **Tool error: second type confirmed** at line 9942: `{"is_error":true,"content":"MCP tool 'sa-git-clone' reported tool error for function call: {...\"isError\":true,\"errors\":[...]}"`. This is a **runtime MCP error** (vs trace_1's permission denial). Both error types come through the same `is_error:true` envelope — translator must parse `content` prefix to distinguish: starts with `"Tool \"X\" is denied:"` → `permission_denied`; starts with `"MCP tool 'X' reported tool error:"` or `"<X> tool error:"` → `runtime_error`.
- ✅ **Batch parallel sub-agent invocation** — line 9267 has 15 tool_use blocks in ONE assistant message (one model turn launched all 15 sub-agents simultaneously). Bridge should NOT assume 1 tool_use per assistant turn.
- ✅ **`todo_write` tool present** (4 calls in this trace, lines like 10298). Optional: span per todo_update, OR flatten to run-level task summary.
- ✅ **`tool_use.input` empty object streamed as `partial_json:"{}"`** (line 10296: `delta.type:"input_json_delta", partial_json:"{}"`). Translator must handle empty-input deltas without crashing.
- ⚠️ **`session_end` always last line** — confirmed (line 12103). No `result` between assistant-final message and `session_end`. Workshop run-finalize signal is unambiguous.

**Decisions (with rationale):**

| # | Decision | Rationale |
|---|---|---|
| D1 | **JSON-stream is the only ingest mechanism.** No `src/server.ts` HTTP listener. | TUI + `--json-file` already gives us real-time full-fidelity events (LLM reasoning, text, tool I/O, sub-agent tree, tokens, duration). HTTP hooks give us only tool I/O. |
| D2 | **FIFO is the primary transport** (regular file + `--resume` mutation is a footgun; FD-passing requires wrapper). | Documented in `qwen --help` as supported target for `--json-file`. Pairs naturally with `readline` on Node side. |
| D3 | **`bridge watch` is auto-spawned by Qwen Code's `SessionStart` hook** (matcher `.*`, type `command`, stdin = JSON event, statusMessage visible in TUI). | Zero per-session setup. `bridge watch` reads `session_id` + `cwd` from stdin, mkfifos `/tmp/agenttrace-qwen/$SESSION_ID.fifo`, then immediately switches to `bridge tail` mode in-process. When Qwen exits, Qwen's session cleanup closes the FIFO; bridge detects EOF and exits. |
| D4 | **Dual CLI surface** — `bridge watch` (stdin-JSON hook entrypoint) and `bridge tail <path>` (manual FIFO/file reader). Same translator underneath. | Lets us unit-test the file reader independently of the hook entrypoint. Also useful for replay from saved JSONL. |
| D5 | **`bridge install` subcommand patches `~/.qwen/settings.json`** to register the `SessionStart` hook (with backup + interactive confirm). Plus documented snippet in `README.md` for manual copy-paste. | (c) best of both: easy for casual users, controllable for advanced users. |
| D6 | **One `agent_provider` discriminator in shipper**: `"qwen_code"` for Qwen, `"gigacode"` for GigaCode. CLI `--provider` flag overrides auto-detection. Auto-detection source **not yet confirmed** against real fixtures — candidate fields: `system/session_start.data.version` prefix (`26.9.64` → gigacode; `0.23.x` → qwen_code), or stdin hook payload's `product_name`. | Per umbrella `openspec/config.yaml` HARD rule "OpenCode-only — never add Codex/Claude/Anthropic-specific code": we add `qwen_code` + `gigacode` providers, both forks of QwenLM/qwen-code, both hook-fork compatible. |
| D7 | **Single JSONL schema for Qwen and GigaCode** (confirmed 2026-09-22 against `trace_1.json`). Both emit `{"type":"system","subtype":"session_start",...}` and `{"type":"system","subtype":"session_end",...}`. Initial guess (from truncated screenshot) that GigaCode used `type:"session_start"` without subtype was wrong. | Required to handle both Kolya's `qwen 0.23.4` and GigaCode fork. If a future Qwen or GigaCode release diverges, F-026 splits this decision. |
| D8 | **TUI-only mode**. No `-y` headless path is planned. Live smoke requires Kolya's hands on a TUI. CI uses fixture files. | Kolya explicit: "В режиме сервера или в безгаловом не планируем запускать". |
| D9 | **HTTP-bridge (`src/server.ts`, `:5898`) deferred indefinitely.** Only the corporate GigaCode fork without TUI would need it — not on Kolya's roadmap. | Don't write code we won't use. Re-add as F-026 only if/when corporate asks. |
| D10 | **Sub-agent visibility default: parent-bridge-only.** The bridge sees each sub-agent invocation as one `tool_use` (start) + `tool_result` (final return) span pair in the parent trace. Sub-agent internals (sub-`tool_use`, sub-thinking, sub-tool-result) are NOT visible. Sub-agent `parent_tool_use_id` semantics and whether sub-agent events live in a separate file (`~/.gigacode/projects/<cwd-hash>/subagents/<sid>/agent-<type>-<id>.jsonl`) are HYPOTHESIS not confirmed — see fixture section "Partial fixture received 2026-09-22" ⚠️ entry. If Kolya verifies sub-agent events do live in separate files, F-026 adds option (a): bridge discovers and tails each new sub-agent file as it appears (requires cwd-hash knowledge + `fs.watch` on the subagents dir). | Minimal-bridge principle; deferred complexity unless requested. |
| D11 | **Reverse-direction: ship to `:5899/v1/` (Workshop) using the OpenCode-plugin wire format** (proposal D3 from F-023 umbrella proposal still holds). | Zero Workshop-side adapter needed beyond the `agent_provider` field already in the F-023 plan. |

**Out of scope (this iteration):**
- Any HTTP server (`src/server.ts`).
- Any `handler-qwen.ts` / `handler-gigacode.ts` files that receive hook events via HTTP.
- OTLP receiver in Workshop (umbrella F-025 — irrelevant once we're JSONL-only).

**Files to create (no code yet — Kolya will send fixtures first):**
- `src/cli.ts` — commander-style bin with subcommands `watch`, `tail`, `install`, `uninstall`.
- `src/session-watch.ts` — `SessionStart` hook stdin reader, mkmfifo, hands off to `tail` mode.
- `src/tail.ts` — readline by line over FIFO/file, zod-validate, call translator.
- `src/translator.ts` — pure function: 1 JSONL event → 1 or 0 Workshop spans.
- `src/wire-format.ts` — zod schemas for both Qwen event shapes (`{type:"system",subtype:"init"}` and `{type:"session_start"}`) and the common internal shape.
- `src/shipper.ts` — POST to `localhost:5899/v1/` with backoff on 429.
- `src/config.ts` — env-var loader (`BRIDGE_PORT` reserved for future HTTP, `WORKSHOP_URL` with `isLocalUrl()` guard from F-005 plugin pattern).
- `src/logger.ts` — leveled logger.
- `src/install.ts` — patches `~/.qwen/settings.json` with backup + dry-run mode.
- `tests/wire-format.test.ts` — round-trip: both Qwen-shape fixtures → zod → common shape.
- `tests/translator.test.ts` — each event type produces correct Workshop-span.
- `tests/tail.test.ts` — synthetic FIFO + assert line-by-line shipper invocation (mocked).
- `tests/shipper.test.ts` — backoff on 429, env-var hardening (no `WORKSHOP_URL=evil.com`).
- `ai-docs/specs/F-025-wire-contract.md` — JSONL event shapes + translator mapping table (replaces the "HTTP wire contract" place from F-023).
- `README.md` — install / `bridge install` / `bridge tail` example / `~/.qwen/settings.json` snippet.
- `.github/workflows/ci.yml` — `tsc --noEmit` + `node --test` against fixtures.

**Todos (waiting on Kolya's fixtures before starting):**
- [ ] **Kolya: send real `qwen.jsonl`** — interactive TUI session, includes 1+ sub-agent invocation and 1+ tool error (`is_error: true` in `tool_result`).
- [ ] **Kolya: send real `gigacode.jsonl`** — same shape requirements, plus if `parent_tool_use_id` nesting differs from Qwen, document it.
- [ ] **Kolya: verify sub-agent event visibility model.** Run a long-running sub-agent in TUI (e.g. one that takes >30s, not a ping-test), and simultaneously tail both `/path/to/main/trace.jsonl` AND `~/.gigacode/projects/<cwd-hash>/subagents/<sid>/agent-*.jsonl`. Report: (i) does the main trace get ANY events with non-null `parent_tool_use_id` while sub-agent runs? (ii) does the sub-agent file exist with JSONL events? (iii) does the sub-agent file have its own `system/session_start` or just bare events? This determines whether we ship option (b) parent-only (default) or add option (a) sub-agent tailed files in F-026.
- [ ] (after fixtures) Write `src/wire-format.ts` zod schemas for both event shapes; round-trip tests green.
- [ ] (after fixtures) Write `src/translator.ts`; unit tests on Kolya's fixtures.
- [ ] (after fixtures) Write `src/tail.ts`; synthetic FIFO test.
- [ ] (after fixtures) Write `src/shipper.ts`; mocked HTTP tests.
- [ ] (after fixtures) Write `src/session-watch.ts`; spawn `tail` in-process.
- [ ] (after fixtures) Write `src/install.ts`; dry-run + real-patch tests.
- [ ] (after fixtures) Write `src/cli.ts` + wire all subcommands.
- [ ] (after fixtures) Write `README.md` install/usage/example.
- [ ] (after fixtures) Write `.github/workflows/ci.yml` (tsc + node --test on fixtures committed to repo).
- [ ] Live smoke (Kolya in TUI): start `bridge tail /tmp/agenttrace-qwen/$SESSION.fifo` in tmux pane B → in pane A run `qwen --json-file /tmp/agenttrace-qwen/$SESSION.fifo` → verify `sqlite3 ~/.raindrop/raindrop_workshop.db "SELECT id, json_extract(metadata,'$.agent_provider') FROM runs ORDER BY last_updated_at DESC LIMIT 3"` shows `qwen_code`.
- [ ] Live smoke (Kolya in TUI): same with `bridge install` first → no manual `bridge tail` invocation, runs go through automatically.

**Cross-repo impact (unchanged from F-023):**
- `agenttrace-opencode-plugin` — NO change.
- `agenttrace` — minimal: read `agent_provider` field, UI source badge + facet (handled by umbrella workshop's F-023 Stages 4-5).

### F-024 — Rename stack: `openworkshop-qwen-bridge` → `agenttrace-qwen-bridge`

**Context:** Companion rename to the umbrella stack rename (F-024 in `opencode-workshop` and `opencode-workshop-plugin`). As of 2026-09-17 this bridge repo is renamed from `openworkshop-qwen-bridge` → `agenttrace-qwen-bridge` to align with the agenttrace identity.

**Files to edit:**
- `package.json` — `name`, `bin`, `description`, `homepage`, `bugs.url`, `repository.url`, `keywords`
- `README.md` — title, install command, description, public package links
- `AGENTS.md` — title, sibling-repo table, public package block, hard-rules wording, Quick Reference
- `ai-docs/PLAN.md` — title + Roadmap + F-023 description + F-024 entry

**Out of scope:**
- `ai-docs/specs/handoff/F-023-corporate-handshake.md` — historical handoff to the corporate GigaCode author dated 2026-09-17. References to `openworkshop-qwen-bridge` paths are part of the historical record and stay intact.
- Remote push (`git remote add origin` + `git push`) — Kolya action.

**Deferred to separate Kolya-authorized steps:**
- `npm publish @grudanov-nikolay/agenttrace-qwen-bridge@<future>`
- `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` (first push to a brand-new remote)

**Todos:**
- [x] `package.json` — name, bin, description, homepage, bugs.url, repository.url, keywords
- [x] `README.md` — header, install command, public package links, install/recipe
- [x] `AGENTS.md` — title, sibling-repo table, public package block, hard-rules wording, Quick Reference, footer
- [x] `ai-docs/PLAN.md` — title + Roadmap + F-023 description + F-024 entry + F-025 architecture flip
- [ ] `git init` + first commit (requires Kolya's «приступай»)
- [ ] `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` + first push (Kolya action)

### F-023 — Bridge for Qwen Code + GigaCode → agenttrace daemon

**Status:** Superseded by F-025 (architecture flip). All Stages 1-3 below to be re-scoped under F-025's file list. Original umbrella proposal remains at `~/workspase/projects/agenttrace-stack/agenttrace/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md` for historical reference.

**Original repo-local scope (kept for audit trail, not active):**
- `src/server.ts` — stdlib HTTP server on `127.0.0.1:5898`, routes `/qwen/hook`, `/gigacode/hook`, `/health`, `/ready`. (Replaced by `src/tail.ts` per F-025 D1.)
- `src/handler-qwen.ts` — Qwen Code `PreToolUse`/`PostToolUse`/`Stop`/`SubagentStop` → translator. (Replaced by `src/translator.ts` reading JSONL events per F-025.)
- `src/handler-gigacode.ts` — placeholder stub for the wire contract (corporate agent fills in real handler against `ai-docs/specs/F-023-wire-contract.md`). (Replaced by shared zod schema in `src/wire-format.ts` per F-025 D7.)
- `src/translator.ts` — pure function, hook event → Workshop-span shape. (Reused as-is in F-025.)
- `src/wire-format.ts` — zod schemas (Qwen input + Workshop output). (Expanded per F-025 D7.)
- `src/shipper.ts` — POSTs translated spans to `localhost:5899/v1/` (re-uses OpenCode-plugin wire format). (Reused as-is in F-025.)
- `src/config.ts` — env-var loader with `isLocalUrl()` guard (F-005 pattern from plugin). (Reused as-is in F-025.)
- `src/logger.ts` — leveled logger. (Reused as-is in F-025.)
- `tests/handler-qwen.test.ts` — 6+ unit tests on translated output. (Replaced by `tests/translator.test.ts`.)
- `tests/wire-format.test.ts` — round-trip zod tests. (Kept and expanded.)
- `tests/shipper.test.ts` — HTTP mocking. (Kept.)
- `ai-docs/specs/F-023-wire-contract.md` — canonical document for the corporate GigaCode author. (Renamed/replaced by `ai-docs/specs/F-025-wire-contract.md` per F-025.)
- `.github/workflows/ci.yml` — `tsc --noEmit` + `node --test`. (Kept.)

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
- [ ] Update umbrella `~/workspase/projects/agenttrace-stack/AGENTS.md` to reference this repo
- [ ] `git init` + first commit (requires Kolya's «приступай»)
- [ ] Publish to npm `0.0.1` (after live smoke test, requires Kolya's `npm publish`)

## Closed Features

_(none yet — this repo is brand new)_

---

*Maintained by Miko (Hermes Agent) under Kolya's direction. Update this file when the repo state changes meaningfully. The 2026-09-18 architecture flip (HTTP-bridge → JSON-stream + SessionStart-hook wrapper) was discussed live with Kolya; F-025 captures the decisions so future sessions can pick up from here without re-litigating.*
