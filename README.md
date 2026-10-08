# agenttrace-qwen-bridge

> **Status:** scaffold only — no code yet. Implementation is blocked on real TUI fixtures (see [`ai-docs/PLAN.md`](./ai-docs/PLAN.md), F-023/F-025). Architecture below is the locked F-025 design (2026-09-18), which superseded the original F-023 HTTP-hook plan.

Bridge that reads **Qwen Code** and **GigaCode** JSONL event streams (their interactive TUI's `--json-file` dual-output mode) and POSTs them, translated into the OpenCode-plugin wire format, to the local **agenttrace** daemon at `http://localhost:5899/v1/`.

## How it works (planned, F-025)

1. `bridge install` patches `~/.qwen/settings.json` to register a `SessionStart` hook (with backup + confirm).
2. On every Qwen Code / GigaCode session start, the hook spawns `bridge watch`; it reads `session_id` + `cwd` from the hook's stdin JSON and creates `/tmp/agenttrace-qwen/$SESSION_ID.fifo`.
3. You run the TUI as usual, adding the dual-output flag: `qwen --json-file /tmp/agenttrace-qwen/$SESSION_ID.fifo` — the TUI renders on stdout while full-fidelity JSONL events stream to the FIFO in real time.
4. The bridge validates each event with a single zod schema (Qwen and GigaCode share the `{"type":"system","subtype":...}` envelope — verified against real fixtures), translates to the OpenCode-plugin wire format, and POSTs to `127.0.0.1:5899/v1/` with backoff on 429.
5. Runs appear in the agenttrace UI tagged by `event_name` (`qwen_code_session` / `gigacode_session`) — the same `All agents` dropdown that already filters OpenCode runs.

`bridge tail <path>` reads any FIFO/JSONL file manually (replay from saved traces); `bridge uninstall` removes the hook.

**Read-only by design:** the bridge never injects context and never blocks tool calls — it only observes and ships spans.

## Why a separate repo

Qwen Code and GigaCode are community tools (forks of Gemini CLI / Qwen Code), not part of Kolya's agenttrace fork. Putting their translation logic into the agenttrace daemon would pollute it with vendor-specific code that the project's `openspec/config.yaml` does not want to own. The bridge runs out-of-process, installable via `npm i -g`, single responsibility. It impersonates the OpenCode-plugin wire format, so the daemon needs zero changes to accept Qwen/GigaCode runs.

## Install (planned)

```bash
npm i -g @grudanov-nikolay/agenttrace-qwen-bridge
bridge install     # registers the SessionStart hook in ~/.qwen/settings.json
```

Full design + open blockers: [`ai-docs/PLAN.md`](./ai-docs/PLAN.md). Original cross-repo proposal (pre-architecture-flip, historical): [`openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`](../agenttrace/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md) in the daemon repo.

## Public package

- **npm:** `@grudanov-nikolay/agenttrace-qwen-bridge` (not yet published)
- **GitHub:** `nikolay-grudanov/agenttrace-qwen-bridge` (not yet pushed)

## License

Dual copyright (c) 2026 Raindrop AI + (c) 2026 Nikolai Grudanov. MIT. See [LICENSE](./LICENSE).

---

*Maintained by Nikolai Grudanov (MIFI, M.Sc. 01.04.02) via Miko (Hermes Agent).*
