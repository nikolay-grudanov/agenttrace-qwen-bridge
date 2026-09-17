# agenttrace-qwen-bridge

> **Status:** scaffold only. No code yet. See [`ai-docs/PLAN.md`](./ai-docs/PLAN.md) for the F-023 roadmap.

Bridge service that translates **Qwen Code** and **GigaCode** HTTP-hook events into the OpenCode-plugin wire format and POSTs them to the local **agenttrace** daemon at `http://localhost:5899/v1/`.

## Why a separate repo

Qwen Code and GigaCode are community tools (forks of Gemini CLI / Qwen Code), not part of Kolya's agenttrace fork. Putting their translation logic into the agenttrace daemon would pollute `agenttrace` (formerly `opencode-workshop`) with vendor-specific code that the project's `openspec/config.yaml` does not want to own. The bridge runs out-of-process, installable via `npm i -g`, single responsibility.

## Install (planned)

```bash
npm i -g @grudanov-nikolay/agenttrace-qwen-bridge
agenttrace-qwen-bridge    # binds 127.0.0.1:5898, forwards to 127.0.0.1:5899
```

## How it works (planned)

1. Receives HTTP-hook events from Qwen Code / GigaCode on `:5898/qwen/hook` and `:5898/gigacode/hook`.
2. Translates them into the same wire format the OpenCode plugin emits to `:5899/v1/`.
3. Tags each forwarded span with `agent_provider` in `runs.metadata`.
4. Returns a passive `{continue: true}` response — never blocks tool calls.

Full design: [`ai-docs/PLAN.md`](./ai-docs/PLAN.md), [`openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`](../opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md) in the umbrella repo.

## Public package

- **npm:** `@grudanov-nikolay/agenttrace-qwen-bridge`
- **GitHub:** `nikolay-grudanov/agenttrace-qwen-bridge`

## License

Dual copyright (c) 2026 Raindrop AI + (c) 2026 Nikolai Grudanov. MIT. See [LICENSE](./LICENSE).

---

*Maintained by Nikolai Grudanov (MIFI, M.Sc. 01.04.02) via Miko (Hermes Agent).*
