# F-023 — Corporate agent handoff prompt (next session)

> **Created:** 2026-09-17 by Kolya.
> **Status:** ready for next session. Do NOT start work without Kolya's explicit «приступай».
> **Where this lives:** `openworkshop-qwen-bridge/ai-docs/specs/handoff/F-023-corporate-handshake.md` (also copied to `~/.hermes/profiles/wsoc_main/prompts/` for safety).

---

## Что я сделал в этой сессии

Заложил каркас нового репозитория `openworkshop-qwen-bridge` и обновил umbrella `AGENTS.md`. Подробнее — в `~/workspase/projects/opencode-workshop-stack/STATUS.md` и `opencode-workshop/ai-docs/PLAN.md` (entry «F-023 — Multi-source agent instrumentation»). Решение зафиксировано в Hindsight (банк wsoc).

**Что точно НЕ сделано:**
- `git init` в `~/workspase/projects/openworkshop-qwen-bridge/`
- Никаких коммитов ни в новом репо, ни в umbrella `AGENTS.md`
- Никакого кода в `src/`, `tests/`, `scripts/`
- Никакого `npm publish`
- Никаких правок в `opencode-workshop/` коде

---

## Контекст, который нужно держать в голове

1. **Проект = Kolya's fork стека `raindrop-ai/workshop`.** Два существующих соседа: `opencode-workshop` (daemon, Bun) и `opencode-workshop-plugin` (Node, npm tarball). Новый третий: `openworkshop-qwen-bridge` (Node, npm).
2. **Цель F-023** — дать Workshop daemon возможность принимать телеметрию от Qwen Code (Alibaba's Gemini-CLI fork) и GigaCode (Sberbank's fork of Qwen Code с вырезанным OTLP). Оба используют только HTTP-хуки. Архитектура = **bridge-only**: новый sibling-repo переводит хуки в OpenCode-plugin wire format и шлёт в `:5899/v1/`. Workshop daemon меняется минимально (читает `agent_provider` поле, добавляет UI badge/facet).
3. **GigaCode** — корпоративный агент на Kolya's ноуте пишет адаптер. У него ОТДЕЛЬНОЕ ТЗ (то, что Kolya пришлёт файлами в этой сессии). Наша задача — изучить ТЗ, выровнять wire contract, заложить инфраструктуру, и только потом «приступай» → разработка Qwen Code-адаптера в `openworkshop-qwen-bridge`.

---

## Что ты получишь от Kolya на входе

Коллегиальный агент пришлёт набор файлов (предположительно):
- ТЗ / спецификация (markdown или PDF)
- Примеры payload'ов GigaCode (JSON-файлы или curl-трейсы)
- Возможно, описание своего wire-contract (если корпоративный агент уже начал его определять)
- Возможно, issue с вопросами / открытые точки

**Все файлы** положи в `~/workspase/projects/opencode-workshop-stack/` (временная папка, почистим после анализа). Затем выполни шаги 1-7 ниже.

---

## Твои действия (по шагам)

### Шаг 1. Прочитай durable state (порядок важен)

Не пропускай ни одного — они образуют цепочку зависимостей:

1. `~/workspase/projects/opencode-workshop-stack/AGENTS.md` — hard rules, umbrella-архитектура (там уже есть раздел про `openworkshop-qwen-bridge`).
2. `~/workspase/projects/opencode-workshop-stack/STATUS.md` — текущее состояние стека, T1-F roadmap entry.
3. `~/workspase/projects/openworkshop-qwen-bridge/AGENTS.md` — внутри-репо гайд.
4. `~/workspase/projects/openworkshop-qwen-bridge/ai-docs/PLAN.md` — F-023 mirror, актуальные todos.
5. `~/workspase/projects/opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md` — канонический дизайн (D1-D13, stages 0-7, scope anchors).
6. `~/workspase/projects/opencode-workshop/openspec/specs/source-aware-ingestion/spec.md` — capability spec (R1-R6, S1-S7, OQ1-OQ4).
7. `~/workspase/projects/opencode-workshop/ai-docs/specs/F-023-multi-source-ingestion-research.md` — research с mapping Qwen Code spans → NormalizedSpan.
8. Hindsight (банк `wsoc`) — последние 30-40 фактов по F-023 (retain через `hindsight_recall`). Там зафиксированы: переименование bridge, GigaCode identity correction, hard-rule соответствие.
9. Скилл `opencode-workshop-development` через `skill_view` — там паттерны lockstep-плагина, F-005 env hardening, live-test recipes.

**Время:** ~5 минут. Не пропускай — это single source of truth, любая правка мимо этих документов = drift.

### Шаг 2. Прочитай присланные файлы

Сначала **составь опись** присланного: `ls -la` папки, `file` для бинарников, `head` для markdown. Затем читай в порядке приоритета:

1. Если есть ТЗ — прочитай целиком, выпиши numbered list «требования».
2. Если есть JSON-примеры payload'ов — каждый открой, посчитай структуру, выпиши в таблицу «event → fields».
3. Если есть open issues — пройди по каждому, сформулируй «наш ответ / наш план».

После прочтения **зафиксируй краткое резюме** (5-10 строк) в Hindsight через `hindsight_retain` с тегом `F-023-corporate-input`. Это страховка от потери контекста при следующих сессиях.

### Шаг 3. Сверь с планом, найди drift

Сопоставь требования корпоративного агента с D1-D13 в `proposal.md` и R1-R6 / S1-S7 в `spec.md`. Конкретно проверь:

| Что сравнить | Где найти | На что смотреть |
|---|---|---|
| Формат payload'ов GigaCode vs наш wire contract | `proposal.md` Stage 1 M1.1 + присланные JSON | Совпадают ли имена полей (`session_id`, `tool_use_id`, `tool_name`, `conversation_id`)? Если расходятся — drift, нужен Stage 0.5 |
| Список событий (PreToolUse / PostToolUse / Stop / SubagentStop) | Qwen Code docs + присланная спецификация GigaCode | Может быть, GigaCode добавляет/убирает события. Например: нет `SubagentStop`, или есть свой `UserPromptSubmit` |
| Поведение «read-only vs inject» | `proposal.md` D10, review Q9 | Корпоративный агент может хотеть, чтобы bridge возвращал `additionalContext` обратно в GigaCode. Это конфликтует с «read-only». Решить с Kolya |
| Порт и URL bridge | `proposal.md` D5, Stage 6.1 | Коллеги могут хотеть другой порт (не 5898) или другой path |
| Авторизация между Qwen/GigaCode ↔ bridge | `proposal.md` Stage 1.1 (упомянут `X-Workshop-Token`) | Если корпоративный агент хочет mTLS / API key / OAuth — это дополнительная работа |
| Persistence в `:5899/v1/` | Qwen Code docs говорят, что у них есть **своё** определение session_id | Может быть, ID коллизия. Решить, какой ID использовать как `runs.convo_id` |

**Drift finding** (если есть) — оформить как **новый sub-stage Stage 0.5: «Reconcile with corporate agent's spec»** в `proposal.md` и `openworkshop-qwen-bridge/ai-docs/PLAN.md`.

### Шаг 4. Сформулируй список вопросов

Если drift найден или есть открытые точки — **НЕ ПРИСТУПАЙ**. Сформируй список вопросов к Kolya, например:

```
[F-023 corporate handshake — questions for Kolya before "приступай"]

1. GigaCode payload field names: drift detected on `tool_use_id` vs `tool_call_id`.
   Corporate spec uses <X>. Qwen Code uses <Y>. Proposal.md assumes <X>. Which to follow?
   (If unsure, follow Qwen Code to keep wire contract uniform.)

2. SubagentStop event: corporate spec says GigaCode does NOT emit SubagentStop
   but uses a different "agent_finished" event. Do we (a) ignore it in the wire
   contract, (b) extend wire contract with custom event, (c) drop subagent support
   for GigaCode entirely?

3. Bridge injection direction: corporate spec asks bridge to return
   `{"additionalContext": "..."}` for some hooks. Our D10 says read-only.
   Do we relax D10 for GigaCode only, or reject this request?

4. ...

5. Auth: no auth header mentioned. Confirmed open bridge on 127.0.0.1 only?
   Or should we add a static shared secret?

6. Path style: corporate uses /api/gigacode/events vs our proposal /gigacode/hook.
   Acceptable, or unify?
```

Передай вопросы Kolya как `kanban_comment` к задаче `t_f76c082a` или новой задаче. Жди ответов.

### Шаг 5. Если drift минимален (≤2 вопросов) — патчи планов согласованным проходом

**Перед патчем прочитай** существующие файлы целиком (через `read_file`, не из memory):
- `~/workspase/projects/opencode-workshop/opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md`
- `~/workspase/projects/opencode-workshop/opencode-workshop/openspec/specs/source-aware-ingestion/spec.md`
- `~/workspase/projects/opencode-workshop/opencode-workshop/ai-docs/specs/F-023-multi-source-ingestion-research.md`
- `~/workspase/projects/opencode-workshop/opencode-workshop/ai-docs/PLAN.md` (F-023 entry + roadmap Tier 1)
- `~/workspase/projects/openworkshop-qwen-bridge/ai-docs/PLAN.md`

**Патчи должны быть согласованы** — если меняешь wire-contract, обнови proposal.md И spec.md И research.md одним проходом. Не плодить дрифт.

**Типичные места для правок:**
- `proposal.md` D11 — wire contract reference (если изменился путь или формат).
- `proposal.md` Stage 1 M1.1 — что именно документирует `ai-docs/specs/F-023-wire-contract.md`.
- `proposal.md` Stages 6/7 — если добавилась новая smoke-проверка.
- `bridge/ai-docs/PLAN.md` — добавить/убрать todos.
- `opencode-workshop/ai-docs/PLAN.md` F-023 — обновить описание, если изменился scope.

После патчей — **сразу** `hindsight_retain` с тегом `F-023-plan-revised`.

### Шаг 6. Когда все вопросы закрыты и планы синхронизированы — жди «приступай»

Коллегиальный агент будет ждать, пока наш pipeline будет готов. Поэтому **до «приступай» никакого кода не пиши**.

После «приступай» начинай с **Stage 1: wire contract**. Конкретно:

1. Создать `openworkshop-qwen-bridge/ai-docs/specs/F-023-wire-contract.md` (Stage 1 M1.1 в proposal.md) — канонический документ.
2. Создать `openworkshop-qwen-bridge/src/wire-format.ts` с zod-схемами (M1.2).
3. Создать `openworkshop-qwen-bridge/tests/wire-format.test.ts` с round-trip тестами (M1.3).

Все три — один commit (или три, по согласованию с Kolya). **Перед commit:**
- `bun install` чисто проходит.
- `bun x tsc --noEmit` exit 0.
- `bun test` зелёный.
- `git status` без untracked (никаких `.zcode/`, `node_modules/`).

**Коммит — только после явного «приступай» / «согласен» / «коммить» от Kolya.** Push — только после явного «пуш».

### Шаг 7. Smoke test и release — НЕ в этой сессии

Дальше пойдёт Stage 2 (translator + shipper + HTTP server), Stage 3 (hardening + CI + первый npm release), Stage 4-5 (workshop daemon patches), Stage 6 (live smoke), Stage 7 (docs + release 0.0.2).

**Всё это — следующие сессии.** В этой (если Kolya не скажет иначе) максимум — закрыть Stage 1 + sync планов.

---

## Жёсткие правила (наследуются из umbrella AGENTS.md)

1. **Никаких auto-commit, auto-push.** `git commit` — только с «приступай» / «согласен» / «коммить» от Kolya. `git push` — только с явным «пуш». НИКОГДА не push сам.
2. **Не перезапускать Workshop daemon.** Если правки в `opencode-workshop/` нуждаются в рестарте — предложить, не выполнять.
3. **Не править `~/.hermes/config.yaml` или `~/.hermes/.env`.** Только предлагать готовые сниппеты.
4. **`ai-docs/PLAN.md` — single source of truth.** Любые правки кода идут paired с правкой PLAN в том же commit.
5. **Один commit = одна Feature-step.** Не объединяй Stage 1 и Stage 2 в один commit.
6. **Не коммить untracked файлы.** `.zcode/`, `node_modules/`, `dist/` — никогда. Перед каждым commit: `git status` проверить.
7. **Bun не жёсткая зависимость** для bridge — пишем на TS/Node. Но используем `bun` для удобства локально (он быстрее).
8. **Файлы от корпоративного агента НЕ коммитим в openworkshop-qwen-bridge.** Они конфиденциальные / не наши. После анализа — удалить или оставить в `/tmp/corporate-handoff/` (вне репо).
9. **Scope OUT остаётся жёстким:** никаких Codex / Claude / Anthropic adapters, никакого upstream sync, никакого cloud.

---

## Контекстные якоря для будущих сессий (быстрый список)

| Что | Где |
|---|---|
| Umbrella AGENTS.md (hard rules) | `~/workspase/projects/opencode-workshop-stack/AGENTS.md` |
| Umbrella STATUS.md (текущее состояние стека) | `~/workspase/projects/opencode-workshop-stack/STATUS.md` |
| F-023 proposal (canonical design) | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md` |
| F-023 capability spec | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/openspec/specs/source-aware-ingestion/spec.md` |
| F-023 research (Qwen span mapping) | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/ai-docs/specs/F-023-multi-source-ingestion-research.md` |
| F-023 entry в workshop PLAN.md | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/ai-docs/PLAN.md` → `### F-023 — Multi-source agent instrumentation (Qwen Code + GigaCode) — Plan only` |
| Bridge repo PLAN.md | `~/workspase/projects/openworkshop-qwen-bridge/ai-docs/PLAN.md` |
| Bridge wire contract (planned Stage 1) | `~/workspase/projects/openworkshop-qwen-bridge/ai-docs/specs/F-023-wire-contract.md` |
| Hindsight memory bank | `wsoc` (use `hindsight_recall` / `hindsight_retain`) |
| Workshop dev skill | `opencode-workshop-development` (load via `skill_view`) |
| Workshop daemon source | `~/workspase/projects/opencode-workshop-stack/opencode-workshop/src/parse.ts` (target for Stage 4 patches) |
| OpenCode plugin wire format (to impersonate) | `~/workspase/projects/opencode-workshop-stack/opencode-workshop-plugin/dist/index.{js,cjs}` (don't edit, just read for reference) |

---

## Конкретные команды, которые понадобятся

```bash
# Чтение текущего состояния
ls -la ~/workspase/projects/openworkshop-qwen-bridge/
cat ~/workspase/projects/openworkshop-qwen-bridge/ai-docs/PLAN.md
cat ~/workspase/projects/opencode-workshop-stack/opencode-workshop/openspec/changes/archive/F-023-multi-source-ingestion/proposal.md | head -100

# После «приступай», Stage 1:
cd ~/workspase/projects/openworkshop-qwen-bridge
bun install
mkdir -p ai-docs/specs
# Создать wire-contract.md
# Создать src/wire-format.ts
# Создать tests/wire-format.test.ts

# Проверки перед commit:
bun x tsc --noEmit
bun test
git status   # никаких untracked
git add <явные пути>
git commit -m "feat(F-023): wire contract + zod schemas"
# НЕ push сам.
```

---

## Финальное напоминание

**До «приступай» — ТОЛЬКО чтение, анализ файлов, патч планов (если drift), сохранение в Hindsight. Никакого кода, никакого `git init`, никакого коммита.**

Когда всё готово и планы синхронизированы — спросить Kolya «готов к Stage 1? жду приступай». После «приступай» — wire contract как первый commit.

---

*Промт составлен Kolya для следующей сессии Hermes Agent в режиме `wsoc_main`. Сохранён:*
- *В этом файле: `openworkshop-qwen-bridge/ai-docs/specs/handoff/F-023-corporate-handshake.md`*
- *Дополнительно: `~/.hermes/profiles/wsoc_main/prompts/F-023-corporate-handshake.md`*
- *Указатель в umbrella `AGENTS.md` Quick Reference: `Bridge handoff prompt | openworkshop-qwen-bridge/ai-docs/specs/handoff/F-023-corporate-handshake.md`*
