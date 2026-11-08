# PLAN.md — agenttrace-qwen-bridge (Kolya's bridge repo)

> **Single source of truth for all development work in this repo.**
> Features at the top (newest first), each with checkboxes. Update in the same commit as the code change.

## Conventions

- **Feature = a vertical slice of work** (one user-visible capability, one bug fix, or one cleanup).
- **Todo = a single atomic step** inside a Feature. Marked `- [ ]` (pending) or `- [x]` (done).
- **F-NNN = Feature ID**, assigned in order of creation. Never reused.
- **Order:** open Feature at the top of the file. Newest F-number first.
- **Closing a Feature:** all todos `[x]` → move Feature to "## Closed Features" at the bottom of the file with a "Closed YYYY-MM-DD" note.

## Corpus — trace fixtures analysed to date (2026-10-20)

| Fixture | Source | Format | Events | What it proved |
|---|---|---|---|---|
| `test_trace_json_file.json` | bridge repo root | `--json-file` wire | 374 | Qwen 0.25.0 wire shape, `protocol_version=2`, `goal_state`, `control_request` |
| `trace_1.json` | private-work-staff/json_ggcode | wire | 5403 | GigaCode 26.9.64, single schema w/ Qwen (D7) |
| `trace_2.json.txt` | hermes attachment | wire | 12104 | `control_request`, MCP error, 15-agent batch, subagent output-file string |
| `tetrace_mini_mcp.jsonl` | private-work-staff/json_ggcode | wire | 18 | mid-stream window (no `stream_event`) |
| `trace_mini_agents_tools.jsonl` | private-work-staff/json_ggcode | wire | 41 | **session_id switch on resume**, async `agent` + `<task-notification>` |
| `trace_3_p{1,2,3}.json` | private-work-staff/json_ggcode | wire | 1500/102/50 | permission_denied shape, GigaCode 26.10.6 |
| `~/.qwen/projects/<slug>/chats/3d983425-….jsonl` | Kolya's machine | **session log** | 196 | 🔑 the transcript format (see F-026) |
| `~/.qwen/projects/<slug>/subagents/3d983425-…/agent-*.jsonl` ×4 + `.meta.json` | Kolya's machine | **session log** | 25/25/31/21 | 🔑 sub-agent internals live in separate files, with parent link |
| `~/.qwen/projects/<slug>/chats/3d983425-….runtime.json` | Kolya's machine | sidecar JSON | — | `pid`, `hostname`, `started_at`, `work_dir`, `qwen_version` |

**Two distinct formats exist.** `--json-file` (the "wire" format) is the *stream* serialization (Anthropic-style `stream_event` deltas, `control_request`, `session_start`/`session_end`). The `chats/*.jsonl` session log is the *persistence* format (Gemini-style `parts`, `parentUuid` DAG, `toolCallResult`, per-sideagent files). F-026 picks the persistence format as the bridge's ingest source. See `ai-docs/specs/F-026-session-log-format.md`.

## Roadmap (Tier 1, next-up) — 2026-10-20

- **F-026 (this). Transport redesign — tail the session log, not `--json-file` + FIFO.** Supersedes D2/D3. See below.
- **F-025.** Wire contract + zod + translator + shipper + `bridge watch`/`tail` + `bridge install`, now **scoped to the F-026 session-log format**. Implementation is unblocked at the schema level; blocked on 4 open questions (bottom of file).
- **F-024 (closed-pending).** Stack rename. `git init` + first commit still pending Kolya's «приступай».

## Active Features

### F-026 — Transport: read the session log (`chats/*.jsonl`) instead of `--json-file` + FIFO

**Context (2026-10-20):** After the 2026-09-18 architecture flip (F-025 D2/D3), the design assumed the bridge must create a FIFO because the `SessionStart` hook cannot tell it where Qwen writes its JSON stream. That assumption is **wrong**. Two discoveries:

1. **Qwen always persists a per-session JSONL session log** at `~/.qwen/projects/<cwd-slug>/chats/<session-uuid>.jsonl`, and the `SessionStart` hook delivers its absolute path on stdin as `transcript_path`. No `--json-file`, no mkfifo, no `--json-fd` wrapper, works identically on `startup` / `resume` / `clear` / `compact`.
2. **The session-log format is strictly richer than the `--json-file` wire format** — see the comparison table below. Choosing the wire format was choosing the worse source.

**Why the session log wins:**

| Capability | `--json-file` wire | `chats/*.jsonl` session log |
|---|---|---|
| `parentUuid` (event DAG → span parenting) | ❌ absent | ✅ every event |
| ISO `timestamp` per event | ❌ absent | ✅ every event |
| Tool result body | `{tool_use_id, is_error, content:string}` — lossy | `toolCallResult{callId,status,executionStatus,resultDisplay}` — **typed union**: `shell_result{exitCode,signal,pid,outcome,output,error,notices,truncated}`, `fileDiff`, `task_execution`, `ask_user_question_answers`, string |
| Token accounting | only in `stream_event.message_start` | `usageMetadata{promptTokenCount,candidatesTokenCount,thoughtsTokenCount,totalTokenCount,cachedContentTokenCount}` + `contextWindowSize` on every assistant turn |
| Sub-agent internals | ❌ not in stream | ✅ `subagents/<sid>/agent-<type>-<toolUseId>.jsonl` + `.meta.json` sidecar |
| Parent→sub-agent link | string in prose | `meta.parentSessionId` + `meta.toolUseId` + `meta.parentAgentId` |
| Session liveness | ❌ | ✅ `chats/<sid>.runtime.json{schema_version,pid,session_id,work_dir,hostname,started_at,qwen_version}` |
| `control_request` / `permission` | ✅ present | ❌ absent |
| Reasoning deltas (`thinking_delta`) | ✅ present | ❌ only finalised `parts[{text, thought:true}]` |
| `model`, `cwd`, `version`, `gitBranch` per event | only in `session_start` | ✅ on every event |
| Noise to filter | low | `system` subtypes `ui_telemetry` (132/196!), `file_history_snapshot`, `attribution_snapshot`, `slash_command` |

**Decisions:**

| # | Decision | Rationale |
|---|---|---|
| **D12** | **Ingest = tail `transcript_path` from the `SessionStart` hook payload.** `bridge watch` reads hook JSON from stdin, resolves `transcript_path`, and tails that regular file incrementally (offset-tracked, tolerant of truncation/rotation). | Removes FIFO + `--json-file` + wrapper entirely. Works on every `source` value. No `/tmp` litter, no dangling FIFOs. |
| **D13** | **The session-log format is the canonical wire contract; the `--json-file` wire format is supported only as a *replay* format** for `bridge tail <fixture>` offline analysis. | Two formats = two zod modules: `src/session-log.ts` (canonical, v1) and `src/wire-jsonl.ts` (best-effort, used by `bridge replay`). Keeps F-023/F-025's fixture work usable without paying for it at runtime. |
| **D14** | **`SubagentStart` / `SubagentStop` hooks are NOT needed.** The bridge derives sub-agent activity by tailing `~/.qwen/projects/<slug>/subagents/<sid>/agent-*.jsonl`. The `subagents/` directory is created **lazily** (verified: 30 of 30 other `~/.qwen/projects/*` dirs have no `subagents/`), so the bridge `fs.watch`es the project dir for `subagents/`, then that dir for `*.jsonl` appearing. Each `.meta.json` gives `parentSessionId` + `toolUseId` + `parentAgentId` + `createdAt`/`lastUpdatedAt`/`status` → exact parent-span attach and duration. | Fewer hooks to register, no reliance on a fork-specific hook event. Verified present in Qwen 0.25.0 on Kolya's machine. |
| **D15** | **Sub-agent `toolCallResult` is `{callId, durationMs}` only** — 41/41 across the 4 fixture files, no output body. So sub-agent tool spans carry name + duration but **no result text**. The sub-agent's final answer is recovered from the parent stream's `<task-notification>` `<result>`. | Don't design around data that isn't persisted. |
| **D16** | **Span correlation key = `callId`**, which is identical in `assistant.message.parts[].functionCall.id` (open) and `tool_result.toolCallResult.callId` (close). Sub-agent spans correlate via `meta.toolUseId`. | Uniform across both formats. |
| **D17** | **The async `agent` tool must NOT be closed by its own `tool_result`.** `toolCallResult.resultDisplay` is `task_execution` and the `functionResponse.output` says `"Background agent launched successfully. task_id: <type>-<toolUseId> … result arrives as a <task-notification> … in a later turn"`. The real close signal is the later `user` event whose `message.parts[0].text` starts with `<task-notification>` and carries `<tool-use-id>`. Both Qwen and GigaCode emit identical text. **Closes the D10 hypothesis.** | Verified in both providers, 4/4 in each fixture set. |
| **D18** | **Filter `system` subtypes.** Drop `ui_telemetry`, `file_history_snapshot`, `attribution_snapshot`; keep `slash_command`. The wire format has no equivalent noise but has `goal_state` (Qwen-only) — also dropped. | `ui_telemetry` is 67% of events in the fixture. |
| **D19** | **`agent_provider` auto-detection = `version` semver major.** `0.x.y` → `qwen_code`; `2x.y.z` (≥ 20) → `gigacode`. Confirmed: Qwen `0.25.0`, GigaCode `26.9.64` / `26.10.6`. `protocol_version` (Qwen 2 / GigaCode 1) is a secondary signal. `--provider` CLI flag overrides. | `system/session_start.data.version` is the only provider signal in the wire format; `version` is on every event in the session log. Version-major is stable and needs no product-name field. |

**Session-log format spec (verified on `3d983425-2f1d-4980-b707-aaeb678b5a78.jsonl`, 196 events):**

```
Event envelope (all types):
  uuid       string UUIDv4        ← dedup key (resume replay re-emits)
  parentUuid string UUIDv4|null   ← DAG edge → span parent
  sessionId  string UUIDv4        ← parent session (subagent files carry the PARENT id)
  timestamp  string ISO-8601 Z    ← duration math
  type       'system'|'user'|'assistant'|'tool_result'
  cwd, version ("0.25.0"), gitBranch

system  + subtype ∈ {slash_command, attribution_snapshot, file_history_snapshot, ui_telemetry}
        + systemPayload (object, shape per subtype)
user    + message{role:'user', parts:[…]}  + promptId ("<sid>########<n>")   ← turn grouping
        + [subtype + systemPayload + deliveredTurn]  ← system-injected user events (e.g. <task-notification>)
assistant + message{role:'model', parts:[…]} + model + usageMetadata + contextWindowSize
        + [agentRound, agentRunId]                    ← sub-agent files only
tool_result + message{role:'user', parts:[{functionResponse:{id,name,response:{output}}}]}
          + toolCallResult {callId, status, executionStatus, resultDisplay}
          + [durationMs]                              ← sub-agent files: callId + durationMs ONLY (D15)

parts item variants:
  {text: string}                          ← plain prose
  {text: string, thought: true}           ← reasoning (finalised; no deltas in this format)
  {functionCall: {id, name, args}}        ← tool call; args fully materialised (no accumulation!)
  {functionResponse: {id, name, response:{output: string}}}   ← echoed inside tool_result

resultDisplay variants (discriminated by .type):
  {type:'shell_result', version, directory, exitCode, signal, pid, outcome,
   outputFiles[], output, text, error, notices[], truncated}
  {type:'fileDiff', fileDiff: string}
  {type:'task_execution', …}               ← the async `agent` launch
  {type:'ask_user_question_answers', …}
  string                                    ← plain text output (read_file etc.)

Sidecars:
  chats/<sid>.runtime.json  {schema_version, pid, session_id, work_dir, hostname, started_at, qwen_version}
  subagents/<sid>/agent-<type>-<toolUseId>.jsonl
  subagents/<sid>/agent-<type>-<toolUseId>.meta.json
     {agentId, agentType, subagentName, description, parentSessionId, toolUseId, parentAgentId,
      createdAt, lastUpdatedAt, status, isBackgrounded, resumeCount, depth, model, resolvedApprovalMode,
      persistedCliFlags{model, authType, baseUrl, approvalMode, maxSubagentDepth:5, …}}
```

**Out of scope (F-026):** any HTTP listener (D1 holds); `control_request` handling (no equivalent in this format — see open question Q4).

**Todos:**
- [ ] Write `ai-docs/specs/F-026-session-log-format.md` — the canonical spec above + translator mapping table + the GigaCode-equivalence column left blank until Q2 lands.
- [ ] `src/session-log.ts` — zod schemas for the 4 event types + `parts` + `resultDisplay` union. All `.passthrough()` (hook/format contracts are forward-extensible).
- [ ] `tests/session-log.test.ts` — parse `chats/3d983425-*.jsonl` + all 4 `agent-*.jsonl` + all 4 `.meta.json`, assert 0 parse failures.
- [ ] `src/subagents.ts` — `fs.watch` the project dir for lazy `subagents/` creation, then that dir for `*.jsonl`; emit sub-agent spans from the `.meta.json` sidecars.
- [ ] `tests/subagents.test.ts` — synthetic dir tree, assert discovery order + parent attach by `toolUseId`.
- [ ] Commit the fixture corpus into `tests/fixtures/` (blocked on Q8 — Kolya's permission).

### F-025 — Architecture: JSON-stream bridge with SessionStart-hook auto-attach

**Context:** 2026-09-18 architecture session with Kolya. Originally F-023 was an HTTP-hook bridge (`:5898`) receiving `PreToolUse`/`PostToolUse`/`Stop`/`SubagentStop`. Smoke-testing revealed (1) Qwen is TUI-only, (2) HTTP hooks need `settings.json` editing + `allowPrivateNetworkHooks`. Kolya proposed `SessionStart`-hook auto-attach for zero per-session setup.

**Status 2026-10-20:** architecture accepted and extended by F-026. All D-decisions below stand **except D2 and D3**, which F-026 replaces with D12/D14. Fixture blockers are resolved for Qwen; 4 questions remain (see bottom of file).

**Fixture 1 — `qwen.jsonl` — ✅ RESOLVED 2026-10-20.**
Received as `test_trace_json_file.json` (203 KB, 374 events, Qwen **0.25.0**, `protocol_version=2`, session `3d983425-2f1d-4980-b707-aaeb678b5a78`, cwd = the bridge repo itself, model `MiniMax-M2.7-highspeed`). Type census: `system`2 / `user`24 / `stream_event`293 / `assistant`46 / `control_request`5 / `control_response`4.
- ✅ `system/session_start` with `data` = `{cwd, protocol_version, session_id, supported_events, version}` **only** — no `tools[]`, no `mcp_servers`, no `agents[]`, no `model`. (`supported_events` = `["system","user","assistant","stream_event","result","control_request","control_response"]`.)
- ✅ `system/session_end` with `data` = `{session_id}` only.
- ✅ **`result` events: 0** — confirms D8/Sharp #13 for Qwen too; run-finalize = `session_end`.
- ⚠️ `stream_event` `goal_state` (1×) — **Qwen-only**, absent from every GigaCode trace.
- ⚠️ Qwen `tool_use.id` = `call_function_<slug>_<n>` (e.g. `call_function_bdmxkuf50vbs_1`), **not** GigaCode's `call_<32 hex>`. Never regex-assume a shape — always key on the literal id string.
- ⚠️ **9 events carry no `session_id` / `uuid` / `parent_tool_use_id`** — exactly the 5 `control_request` + 4 `control_response`, whose key sets are `{type,request,request_id}` and `{type,response}`. zod must make those three fields optional.
- ✅ Qwen `assistant.message` has `stop_reason` + `usage` (GigaCode does not). Qwen `user.message` = `{role, content}`.
- ✅ Qwen `thinking` blocks carry `signature`.
- Tool inventory seen: `agent`(4), `run_shell_command`(3), `tool_call`(3 — MCP proxy, 3 in one message), `read_file`(2), `glob`(1), `edit`(1), `ask_user_question`(1), `tool_search`(1).
- ✅ `<task-notification>` present 4× — with `<remaining>`/`<all-terminal>` but **no `<output-file>`** (GigaCode's variant has it). Task-id prefix `Explore` = the default `subagent_type` when omitted.

**Fixture 2 — `gigacode.jsonl` — ✅ RESOLVED** (5 files, sessions `6cc302b9…`, `7f7459ef…`, `95698ef7…`, GigaCode **26.9.64** and **26.10.6**, model `vip/GLM-5.3`, `protocol_version=1`).
- ✅ D7 holds hard: identical `session_start`/`session_end` envelope and identical `data` key set as Qwen.
- ✅ `result` events: 0 across all 5403 + 12104 + 18 + 41 + 1652 lines.
- 🔑 **`session_id` switches mid-stream on resume.** In `trace_mini_agents_tools.jsonl`, events 1–30 and the trailing `session_end` carry `95698ef7…` while events 31–40 carry `7f7459ef…` (the *original* session). → Group by the **envelope** session_id; **dedup replayed history by per-event `uuid`**.
- 🔑 **Async `agent` + `<task-notification>`** (see D17). GigaCode's notification includes `<output-file>` and `<usage><total_tokens>`; Qwen's does not.
- ✅ `agent` tool input = `{description, prompt, subagent_type}`; `subagent_type` free-form (`Explore`, `general-purpose`, 12 `sa-developer-*` types, `statusline-setup`). 15 `agent` blocks in ONE assistant message → never assume 1 tool_use/turn.
- ✅ Error taxonomy (3 shapes): `permission_denied` = `Tool "X" is denied: …`; `runtime_error` = `Command: …\n<stderr>` (bash non-zero); `mcp_error` = `MCP tool 'X' reported tool error for function call: {…"isError":true…}`.
- ✅ `user` events carry genuine user interjections (`content:[{type:"text"}]`) — must not be mistaken for tool results.
- ⚠️ `tetrace_mini_mcp.jsonl` starts mid-stream with **zero `stream_event`** — truncated windows lose deltas. The session-log format has no deltas at all, so this failure mode disappears under F-026.

**Hook research (Qwen Code 0.25.0, authoritative doc on disk):**
`~/.npm-global/lib/node_modules/@qwen-code/qwen-code/bundled/qc-helper/docs/features/hooks.md` (1562 lines).
- **16 events:** `PreToolUse`, `PostToolUse`, `PostToolUseFailure`, `UserPromptSubmit`, `SessionStart`, `SessionEnd`, `SessionDelete`, `MessageDisplay` (fires repeatedly as the reply streams), `Stop`, `SubagentStart`, `SubagentStop`, `PreCompact`, `Notification`, `PermissionRequest`, `PermissionDenied`, `TodoCreated`, `TodoCompleted`.
- **4 executors:** `command` (JSON on stdin, JSON on stdout), `http` (JSON POST body), `function` (in-process JS, session scope only), `prompt` (LLM evaluation).
- Command-hook fields: `type`, `command`, `name`, `description`, `timeout` (ms, default 60000), **`async`** (background), `env`, `shell`, **`statusMessage`** (rendered in the TUI).
- **Common input payload: `{session_id, transcript_path, cwd, hook_event_name, timestamp}`** + per-event fields; sub-agent context adds `agent_id` + `agent_type`. `SessionStart` adds `{permission_mode: default|plan|auto_edit|yolo, source: startup|resume|clear|compact, model}` — **no json-file/fifo path**. `SessionEnd` adds `{reason: clear|logout|prompt_input_exit|bypass_permissions_disabled|other}`.
- Exit codes: `0` = success (stdout parsed as JSON), `2` = **blocking error** (stderr → model), other = non-blocking. Output fields: `continue`, `stopReason`, `suppressOutput`, `systemMessage`, `decision`/`reason`, `hookSpecificOutput{hookEventName, additionalContext}`.
- **HTTP hooks allow loopback by default** — SSRF protection explicitly permits `127.0.0.1` / `::1`; `security.allowPrivateNetworkHooks` is only needed for `10.x` / `172.16/12` / `192.168.x`. So a loopback HTTP hook needs no settings hack. (Not used — D1 — but it removes the friction that originally killed F-023.)
- Global kill switch: top-level `"disableAllHooks": true`. Contract is forward-extensible ("consumers should ignore unknown fields") → all zod schemas `.passthrough()`.
- Local layout confirmed: `~/.qwen/projects/<cwd-slug>/chats/<sid>.jsonl` (+ `.runtime.json`), `~/.qwen/projects/<cwd-slug>/subagents/<sid>/agent-*.jsonl` (+ `.meta.json`). `~/.qwen/settings.json` currently has **no `hooks` key**. `~/.qwen/sessions/` exists but is empty. No `~/.gigacode/` locally (corporate only).

**Decisions (D2/D3 superseded by F-026; the rest stand):**

| # | Decision | Status |
|---|---|---|
| D1 | JSON-stream is the only ingest mechanism. No `src/server.ts` HTTP listener. | ✅ stands (stream = session log per D13) |
| D2 | FIFO is the primary transport. | ❌ **superseded by D12** |
| D3 | `bridge watch` auto-spawned by the `SessionStart` hook, mkfifo `/tmp/agenttrace-qwen/$SESSION_ID.fifo`. | ❌ **superseded by D12** (auto-spawn still true; mkfifo dropped) |
| D4 | Dual CLI: `bridge watch` (hook entrypoint) + `bridge tail <path>` (manual), same translator underneath. | ✅ stands — add `bridge replay <wire.jsonl>` (D13) |
| D5 | `bridge install` patches `~/.qwen/settings.json` to register the `SessionStart` hook (backup + interactive confirm); README snippet for manual paste. | ✅ stands |
| D6 | One `agent_provider` discriminator: `qwen_code` / `gigacode`; `--provider` overrides. | ✅ stands, detection rule now specified in **D19** |
| D7 | **Single JSONL schema for Qwen and GigaCode.** Both emit `{"type":"system","subtype":"session_start"}` and `session_end`. | ✅ **re-confirmed 2026-10-20** on GigaCode 26.10.6 |
| D8 | TUI-only mode; no headless path. Live smoke requires Kolya; CI uses fixtures. | ✅ stands |
| D9 | HTTP-bridge `:5898` deferred indefinitely. | ✅ stands — and D12 makes it moot |
| D10 | Sub-agent visibility default: parent-only; separate-file tailing was a HYPOTHESIS. | ✅ **HYPOTHESIS CONFIRMED → promoted to D14** (separate files exist, with a `.meta.json` parent link) |
| D11 | Ship to `:5899/v1/` (agenttrace daemon) using the **OpenCode-plugin wire format**. Zero daemon-side adapter beyond `agent_provider`. | ✅ stands |
| D12–D19 | *(see F-026)* | 🆕 |

**Out of scope (this iteration):** any HTTP server; `handler-qwen.ts` / `handler-gigacode.ts`; an OTLP receiver in agenttrace.

**Files to create (no code yet — repo is still scaffold-only):**
- `src/cli.ts` — bin with `watch`, `tail`, `replay`, `install`, `uninstall`.
- `src/session-watch.ts` — `SessionStart` stdin reader → resolve `transcript_path` → hand off to tail in-process; exits on `SessionEnd` hook or `pid` death.
- `src/tail.ts` — offset-tracked incremental file reader → zod → translator.
- `src/session-log.ts` — canonical zod schemas (F-026).
- `src/wire-jsonl.ts` — best-effort zod schemas for the `--json-file` replay format.
- `src/subagents.ts` — lazy-`subagents/` watcher + `.meta.json` parsing.
- `src/translator.ts` — pure fn: 1 session-log event → 1..n spans (stateful across a turn: the `agent` span stays open until its `<task-notification>`).
- `src/shipper.ts` — POST `localhost:5899/v1/`, backoff on 429.
- `src/config.ts` — `WORKSHOP_URL` + `isLocalUrl()` guard (F-005 plugin pattern).
- `src/logger.ts`, `src/install.ts`.
- `tests/{session-log,translator,tail,shipper,subagents,install}.test.ts`; `.github/workflows/ci.yml`.

**Todos (order matters — F-026 items first):**
- [ ] Answer Q1–Q4 below (Kolya's data / corporate answers).
- [ ] `src/session-log.ts` + `tests/session-log.test.ts` green against the on-disk corpus.
- [ ] `ai-docs/specs/F-026-session-log-format.md`.
- [ ] `src/subagents.ts` + `tests/subagents.test.ts`.
- [ ] `src/translator.ts` — spans for: user turn, assistant reasoning, assistant text, tool call/result, **deferred `agent` span closed by `<task-notification>`**, run-level totals. Unit tests on the on-disk corpus.
- [ ] `src/tail.ts` — synthetic file-append test proving incremental read + dedup-by-uuid.
- [ ] `src/shipper.ts` — mocked HTTP, 429 backoff, `WORKSHOP_URL=evil.com` rejected.
- [ ] `src/session-watch.ts` — stdin parse → tail handoff.
- [ ] `src/install.ts` — dry-run + real patch of `~/.qwen/settings.json`, backup, idempotent.
- [ ] `src/wire-jsonl.ts` + `bridge replay` — best-effort, non-blocking.
- [ ] `src/cli.ts` wiring.
- [ ] `README.md`; `.github/workflows/ci.yml`.
- [ ] Commit the fixture corpus to `tests/fixtures/` (needs Q8).
- [ ] **Live smoke A (Kolya, TUI):** `bridge tail ~/.qwen/projects/<slug>/chats/<sid>.jsonl` in tmux pane B → work in pane A → verify `sqlite3 ~/.raindrop/raindrop_workshop.db "SELECT id, json_extract(metadata,'$.agent_provider'), span_count FROM runs ORDER BY last_updated_at DESC LIMIT 3"`.
- [ ] **Live smoke B (Kolya, TUI):** same after `bridge install` — zero manual invocation, sub-agent spans appear nested under their `agent` tool span.
- [ ] **Live smoke C (Kolya, corporate):** repeat A on the GigaCode machine (needs Q1/Q2 answers).

**Cross-repo impact (unchanged):** `agenttrace-opencode-plugin` — **no change**. `agenttrace` — minimal: read `agent_provider`, UI source badge + facet.

### F-024 — Rename stack: `openworkshop-qwen-bridge` → `agenttrace-qwen-bridge`

**Context:** Companion rename to the umbrella stack rename (F-024 in `opencode-workshop` and `opencode-workshop-plugin`). As of 2026-09-17 this bridge repo is renamed to align with the agenttrace identity.

**Files to edit:** `package.json` (name/bin/description/homepage/bugs/repository/keywords), `README.md`, `AGENTS.md`, `ai-docs/PLAN.md`.

**Out of scope:** `ai-docs/specs/handoff/F-023-corporate-handshake.md` — archived to `ai-docs/specs/archive/` (superseded by F-025/F-026). Remote push — Kolya action.

**Deferred to separate Kolya-authorized steps:** `npm publish @grudanov-nikolay/agenttrace-qwen-bridge@<future>`; `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` + first push.

**Todos:**
- [x] `package.json` — name, bin, description, homepage, bugs.url, repository.url, keywords
- [x] `README.md` — header, install command, public package links, install/recipe
- [x] `AGENTS.md` — title, sibling-repo table, public package block, hard-rules wording, Quick Reference, footer
- [x] `ai-docs/PLAN.md` — title + Roadmap + F-023 description + F-024 entry + F-025 architecture flip
- [ ] `git init` + first commit (requires Kolya's «приступай»)
- [ ] `gh repo create nikolay-grudanov/agenttrace-qwen-bridge --public` + first push (Kolya action)

### F-023 — Bridge for Qwen Code + GigaCode → agenttrace daemon

**Status:** Superseded by F-025 → F-026. Kept for audit trail. Original umbrella proposal: `~/workspase/projects/agenttrace-stack/agenttrace/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`.

**Original repo-local scope (superseded):** `src/server.ts` `:5898` HTTP listener, `handler-qwen.ts`, `handler-gigacode.ts`, `translator.ts`, `wire-format.ts`, `shipper.ts`, `config.ts`, `logger.ts`, `handler-qwen.test.ts`, `F-023-wire-contract.md`.

**Cross-repo impact:** no changes to `agenttrace-opencode-plugin`; minimal changes to `agenttrace` (read `agent_provider`, UI badge).

**Todos (Stage 0 repo bootstrap):**
- [x] Create repo (originally `openworkshop-qwen-bridge`, renamed 2026-09-17 F-024)
- [x] `package.json` (npm namespace `@grudanov-nikolay/agenttrace-qwen-bridge`)
- [x] `tsconfig.json` (strict, ES2022), `.gitignore`, `LICENSE`, `README.md`, `AGENTS.md`, `ai-docs/PLAN.md`
- [ ] Update umbrella `~/workspase/projects/agenttrace-stack/AGENTS.md` to reference this repo
- [ ] `git init` + first commit (requires Kolya's «приступай»)
- [ ] Publish to npm (after live smoke, Kolya action)

## Open questions — data needed from Kolya (2026-10-20)

**Q1 (blocking, corporate).** Does GigaCode 26.10.6 also persist `~/.gigacode/projects/<slug>/chats/<sid>.jsonl` in the **same** session-log format, or only the `--json-file` stream? → If yes: **one** translator, F-026 covers both providers. If no: F-027 splits into two translators.
**Q2 (blocking, corporate).** Does GigaCode support `SessionStart` hooks, and does the payload carry `transcript_path`? If not, the corporate install falls back to `--json-file <fifo>` (old D2/D3).
**Q3 (nice-to-have, corporate).** Do GigaCode sub-agent files have the `.meta.json` sidecar with `parentSessionId` / `toolUseId` / `parentAgentId`?
**Q4 (blocking for F-013 parity).** How does a **permission denial / tool failure** appear in the *session-log* format? Every one of the 16 `toolCallResult` in the Qwen fixture is `status:"success"`. Need any real chat log containing `status:"error"` / `executionStatus:"error"` / `shell_result.error != null` / `exitCode != 0`, plus the `control_request`-denied path (which does not exist in this format at all).
**Q5 (verification, 2 minutes).** Confirm that `transcript_path` in the actual `SessionStart` hook payload really points at `~/.qwen/projects/<slug>/chats/<sid>.jsonl`. Drop this in `~/.qwen/settings.json` and run one TUI session:
```json
{"hooks":{"SessionStart":[{"matcher":"*","hooks":[{"type":"command","name":"at-probe","command":"tee /tmp/at-hook.json","statusMessage":"agenttrace probe"}]}]}}
```
Then `cat /tmp/at-hook.json`.
**Q6 (verification, 1 minute).** Confirm an `async: true` command hook does not block the TUI and that its stdout is ignored. Same snippet with `"async": true` and `"timeout": 5000`.
**Q7 (nice-to-have).** A **nested** sub-agent session (`depth ≥ 1`, `parentAgentId != null`) to validate recursive attach — `persistedCliFlags.maxSubagentDepth` is 5, but all four captured sub-agents are `depth: 0`.
**Q8 (needed for CI).** OK to commit the fixture corpus into `tests/fixtures/`? Concretely: `test_trace_json_file.json` (203 KB), `~/.qwen/projects/-home-gna-workspase-projects-agenttrace-qwen-bridge/chats/3d983425-*.jsonl` (399 KB), its 4 `agent-*.jsonl` (27–98 KB) + 4 `.meta.json`, and the 5 GigaCode files from `private-work-staff/json_ggcode/`. Redact or confirm as-is.
**Q9 (nice-to-have).** A **tool-error** Qwen chat log and a `thinking`-heavy log, so the reasoning-span path and the F-013 `status=ERROR` mapping get real fixtures rather than synthetic ones.

## Closed Features

_(none yet — this repo is brand new)_

---

*Maintained by Miko (Hermes Agent) under Kolya's direction. 2026-09-18 architecture flip (HTTP-bridge → JSON-stream + SessionStart hook) recorded in F-025. 2026-10-20 corpus analysis (Qwen 0.25.0 wire + session log, GigaCode 26.9.64 / 26.10.6 wire, full hooks doc) recorded in F-026 — the ingest source moved from `--json-file` to the session log, which is strictly richer and needs no FIFO.*