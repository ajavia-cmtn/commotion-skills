---
name: commotion-generate-scenarios
description: >-
  Build a realistic test set for a Commotion worker — the simulated callers (personalities) and the
  scenarios they run — so the worker can be evaluated automatically. Ground in the live scenario
  schema, design personas and scenarios from the worker's goal, AI-generate scenarios (or author them
  manually / from a real call), and create them on the platform with each write approved. Use this
  whenever the user wants to test, simulate, stress-test, or "make scenarios / personas / a test set"
  for a worker — e.g. "generate test cases for my renewal bot", "make an angry-caller persona and run
  it against the worker". This is step 2 of the quality loop (create-worker → **generate-scenarios** →
  run-evals → improve-worker). Calls the Commotion backend through the thin Commotion MCP server (OAuth — no
  API key in the transcript).
allowed-tools: Read, AskUserQuestion, mcp__commotion-dev3__commotion_request, mcp__commotion-tcuat__commotion_request, mcp__commotion-tcprod__commotion_request, mcp__commotion-dev3__commotion_schema, mcp__commotion-tcuat__commotion_schema, mcp__commotion-tcprod__commotion_schema, mcp__commotion-dev3__commotion_analyzer, mcp__commotion-tcuat__commotion_analyzer, mcp__commotion-tcprod__commotion_analyzer
---

# Commotion: Generate Scenarios & Personalities

Turn a worker's goal into a **test set** that exercises it: a set of **scenarios** (each a simulated
conversation with a goal the worker must achieve) driven by **personalities** (the simulated caller's
persona, voice, and behaviour). You supply the judgment — which personas the domain needs, which
happy/failure/jailbreak paths matter, what each scenario's success looks like — and you make the
platform I/O through the connected **Commotion MCP** server's two tools (`commotion_request` /
`commotion_schema`). This skill carries the endpoints, and you fetch request schemas live from the
OpenAPI spec. **Every write is shown to the user and approved before it happens.**

**Auth is automatic (browser login).** The Commotion MCP handles auth via OAuth: on first use it
opens a Commotion login in the browser, then attaches the user's token to every `commotion_request` /
`commotion_schema` call for you — the token never enters the conversation. Never ask the user for an
email/password or a token, and don't pass a `token` argument. If the two tools aren't available at
all, the MCP isn't connected — ask the user to add/authorize it via `/mcp`.

This is **step 2 of the worker quality loop**:

```
create-worker → [generate-scenarios] → run-evals → improve-worker
                                          └──────── repeat until pass-rate ≥ threshold ────────┘
```

Scenarios and personalities created here are the input to `commotion-run-evals` (step 3), which runs
them as a simulation and reports the pass-rate. The whole loop runs against a specific worker
**version** — see the version rule below.

## Pick the environment before the first tool call

Commotion runs in several environments — dev3, tcuat, tcprod — and **each one has its own connector
exposing its own copy of these tools**. They are separate backends holding different data, and one of
them is **production**. Picking the wrong one is the most damaging mistake available in this skill,
and nothing downstream will catch it: the guarantee is this instruction, not an enforcement.

**Read the environment off the tool name.** Tools are namespaced by the connector they came from, and
the environment appears in that name — `dev3`, `tcuat`/`UAT`, `tcprod`/`Prod`. The exact shape depends
on where you are running: a plugin-declared server name in Claude Code
(`mcp__commotion-tcuat__commotion_request`), or the administrator's connector name in the Claude
desktop app (`mcp__claude_ai_Commotion_Agent_UAT__commotion_request`). **Match on the environment
word, not on a fixed prefix**, and never assume a default — least of all dev3.

1. **The user's words win.** If they name an environment, use the connector whose name identifies it.
2. **Exactly one connector available → use it**, no question needed.
3. **Several available and the user hasn't said → ask once**, listing the environments you can see.
   Then use that one connector for **every** call for the rest of the task. Never mix connectors
   within a task.
4. **If no tool name clearly identifies an environment, ask rather than guess.** Name the connectors
   you can see and let the user choose.
5. **State which environment you are working in** in your first substantive reply — and again,
   plainly, before the first write to a production connector.
6. **Never retry a failed call on a different connector.** A `401`/unauthenticated error means that
   connector has not been authorized — tell the user to connect it in their client's connector
   settings. It never means "try another environment".

Where this file names `commotion_request`, `commotion_schema` or `commotion_analyzer` without a
prefix, it means that tool **on the connector you selected here**.

## Prerequisites (verified live — the test set is only runnable if these hold)

- **Any channel works — voice, chat, or structured output.** All three simulate through the same
  `POST /simulation/run`; set each scenario's `aiAgentChannelType` to match the agent (`VOICE` for
  `VOICE_AGENT`, **`CHAT` for both `CHAT_AGENT` and `STRUCTURED_OUTPUT`**). Verified live 2026-08-03:
  chat worker → `passRate 100.0`, SO worker → `PASS`. Do **not** convert a chat worker to voice just to
  test it, and do not fall back to a live `/aiworker/run` session for chat/SO.
- **The worker must have been deployed (live) at least once.** AI scenario-generation and simulation
  both need a live runtime — for a **never-deployed** worker, `POST /scenario/generate` produces
  **nothing** and sims fail with *"Worker is not available"*. (A draft *version* of an already-live
  worker is fine.) This — not the channel — is the real prerequisite.
- **Personalities must be voice-enabled** (`voiceEnabled:true` + a voice) to drive **voice** simulations;
  for **chat/SO** scenarios `voiceEnabled:false` is correct and sufficient.

## When to use this

The user wants to build test cases / scenarios / simulated callers for a worker, or asks to "test" or
"simulate" a worker before/after deploying. If they then want to actually run them and read scores,
that's `commotion-run-evals`; if they want to iterate the worker until it passes, that's
`commotion-improve-worker`. You usually arrive here already knowing the worker id (from create-worker);
if not, ask for it or list `GET /aiworker`. **For the whole build → test → improve pipeline in one
request, use the `commotion-quality-loop` orchestrator** (it invokes this skill as its scenario step).

## How this skill talks to the platform (read first)

All platform I/O goes through the connected **Commotion MCP** server — two tools, no scripts, no keys.
It's the **one unified backend**: the scenario / simulation / eval endpoints live in the same OpenAPI
spec as `/aiworker` and `/aiagent`, so nothing about auth or schema-fetching changes:

- **`commotion_request`** — one authenticated call: `{ "method": "GET|POST|PUT|DELETE", "path": "/…",
  "body": <JSON, for writes> }` → `{ "status", "body" }` (a non-2xx is **returned, not thrown** —
  read it and adjust). Pass a **path** (the base URL is fixed server-side) and the request payload as
  `body` — no temp files.
- **`commotion_schema`** — a bundled request schema: `{ "schema_name": "GenerateScenarioRequest" }` →
  the JSON Schema with its `$defs`. Any component name in the live spec works. **Never invent a field
  that isn't in the schema.**

**Three rules, all absolute:**

- **Never discover a shape by writing.** `commotion_schema` (plus `GET`s) is how you learn a body.
  A `POST`/`PUT` carrying placeholder text to "see what sticks" is not a probe — `PUT` is a **full
  replace**, so it overwrites the real record with your placeholder. If you must try a write shape,
  create a throwaway `ZZ-TEST-…` worker, probe there, and delete it.
- **A client-side denial is not an API error.** *"Permission for this action was denied"* /
  *"Blocked by classifier"* / a permission prompt comes from the **client**, not the backend. Never
  retry it, never switch connectors, never re-route it through `Bash`/`curl`, and **never hand the user
  a script plus a token to run the call themselves** — that leaks the credential the MCP exists to hold,
  skips the audit log, and is built on a base URL you had to guess. Staging the *content* in a file is
  fine; the write still goes through `commotion_request` on approval. Re-read your body (the denial is
  usually right), then tell the user plainly what was blocked and move on to the unblocked work.
- **Never claim "the API doesn't expose that" from a path you guessed, and never answer a question
  about one environment with another environment's data.** Tools and knowledge hang off the **worker**
  (`GET /ai-worker-tool?aiWorkerId=&version=`, `GET /aiworker/knowledge?aiWorkerId=`), never off the
  agent. Check the endpoint map and the live spec, then state what you checked. If a read is
  unavailable in the selected environment, say it is unavailable — don't substitute dev3 for tcuat.

*(Full detail: `references/api-and-auth.md` in **commotion-create-worker**.)*


**Auth is automatic — there is no key.** The MCP client owns OAuth: the first time the
Commotion MCP is used it opens a Commotion login in the browser, then attaches the user's token to
every call. Never ask the user for a token. If the two tools aren't available, the Commotion MCP
isn't connected — ask the user to add/authorize it (in Claude Code: `/mcp` → **commotion** →
Authenticate), then continue.

**Read ids from results, not `jq`:** `commotion_request` returns the parsed `body` — after a create
the new id is `body.id`; from a list it's `body[0].id`; the async scenario-generate returns
`body.scenarioGenerationId`. Feed that value into the next call's `path`.

The endpoint map, error semantics, and schema-name list for the scenario/personality/eval domain are
in `references/eval-domain-api.md` — the single "how to call it" reference. Field *shapes* always come
from `commotion_schema`; the reference files are the *behavior* the schema doesn't tell you. Detailed
scenario/personality recipes are in `references/scenarios-and-personalities.md`.

**Execution rules:** one phase at a time, in order; read the reference named by a phase before acting;
show every write before you make it.

## Phase 0 — Ground yourself in the real schema (always, before drafting)

Never invent field names or values. Read the contracts from the server first:

1. `commotion_schema` `{ "schema_name": "GenerateScenarioRequest" }`, `{ "schema_name":
   "ScenarioRequest" }`, and `{ "schema_name": "PersonalityRequest" }` → the exact bodies (bundled
   with `$defs`).
2. `commotion_request` `{ "method": "GET", "path": "/scenario/dropdown-config" }` → the **valid
   values** for `complexity`, `pathType`, `scenarioGenerationType`, `channelType` (each a
   `{code,label,isDefault}`) **plus `maxScenarioGenerationLimit` and `maxScenarioRunLimit`** — respect
   these limits.
3. `commotion_request` `{ "method": "GET", "path": "/scenario/intent-values" }` → existing intent tags
   (typeahead).
4. `commotion_request` `{ "method": "GET", "path": "/aimodel?pageSize=200" }` → valid provider/model
   codes for the **simulator LLM** (`LLMConfig` on generate + run — the LLM that powers scenario
   generation and the simulated caller). ⚠ **Pass `pageSize`** — the endpoint paginates at ~10 rows and
   the bare call returns 10 junk `azure_openai` test rows with **zero Commotion models** (verified live
   2026-08-06), which reads falsely as "no Commotion models here."

## Phase 1 — Identify the target worker + version  ·  HUMAN INPUT (only what's missing)

The whole test set is scoped to **one worker and one version**:

- **`aiWorkerId`** — usually carried over from create-worker; else ask or `GET /aiworker` (list).
- **`version`** — **which version are you testing?** In the quality loop you test the *draft* you're
  improving; for a one-off check of a deployed worker you test the live version. Default to the version
  the user is iterating on. (Scenarios are created with this `version` in the body.)
- **Channel** — `aiAgentChannelType`, taken from the target agent's type: `VOICE` for a `VOICE_AGENT`,
  **`CHAT` for a `CHAT_AGENT` *or* a `STRUCTURED_OUTPUT` agent** (there is no SO channel value). Set it
  explicitly on every scenario — it's what makes the run execute on the right channel. Confirm by reading
  back `channelTypeLabel` (the **Type** column in the Scenarios UI).
- Decide whether to test the **whole worker** or a **specific agent** (`aiAgentId` +
  `isTestSpecificAgent: true`) — useful for a multi-agent worker when you want to test one specialist.

**Version rule (important — verified shape):** list endpoints filter by `aiWorkerId`, **not** by
`version` (there is no `version` query param on `GET /scenario`). Each `ScenarioResponse` carries its
own `version`. So generate/create at the version under test, and when listing, **read each scenario's
`version` field** to know which version it belongs to. See `references/scenarios-and-personalities.md`.

## Phase 2 — Design the personalities (the simulated callers)

A scenario runs against a **personality** — the persona, mood, voice, and behaviour of the simulated
caller. Design the personas **this domain actually faces**, not a generic set: the cooperative caller,
the frustrated/angry caller, the impatient interrupter, the code-switching (e.g. Hinglish) caller, the
caller on a noisy line, the adversarial/jailbreak caller. Each persona is reusable across scenarios.

- **Reuse** what's there first: `GET /personality` (filter by `gender`/`mood`/`voiceEnabled`/`searchText`).
  **⚠ Check `voiceEnabled` on the record you pick — do not assume it.** Verified live 2026-07-28: **all
  ten personalities on dev3 had `voiceEnabled: false`**, so reusing one silently produces a **mute
  caller** — the simulation runs, bills real minutes, and every call comes back
  `userTurnCount: 0` / `userAudioSeconds: 0.0` / `stopReason: user_idle_timeout` with the worker
  politely greeting an empty line. The pass-rate from such a run is meaningless. If nothing on the list
  is voice-enabled, **create one** (below) rather than reusing. Also note a personality's *name* may
  describe a TTS voice (`Eleven_Flash_English`, `Cartesia_Flash_English`) rather than a caller behaviour
  (`Angry`, `Frusted`, `2x Speaking`) — read `prompt`, `mood`, `interruptionLevel` and `voiceEnabled`,
  never the name.
- **AI-draft the persona prompt**: `POST /personality/prompt/generate` `{description}` → returns
  `{generatedPrompt}`. Edit it, then create the persona.
- **Create**: `POST /personality` (`PersonalityRequest` — `name, gender` (MALE/FEMALE), `mood`
  (HAPPY/FRUSTRATED/…), `prompt`). **For voice simulations set `voiceEnabled:true`** + a voice
  (`voiceProvider/voiceModel/voiceId/languages`) — without it the sim has no caller audio. **For chat
  and structured-output simulations, `voiceEnabled:false` is correct** — those channels have no audio,
  and the voice-enabled requirement above does not apply (verified live: passing chat + SO runs both
  used a `voiceEnabled:false` persona). Reuse the worker's own voice, picking the `voiceId` from
  `GET /aiworkervoice?providerId=commotion-tts&modelId=commotion-laya-v1-5` (verified good:
  `d6d81480-227c-41cd-af4e-f483262cef0b` — the voice **Poornima** — which covers en + hi and more).
  Keep `voiceProvider` a **`commotion-*`** provider: a non-Commotion voice needs a
  `voiceProviderCredentialId` you cannot obtain over the API. Realism dials:
  `speakingSpeed`, `interruptionLevel`, `backgroundNoise` (e.g. `NONE`), `packetLoss`. For a
  **bilingual / code-switching** persona set `languages:["en","hi"]` and describe the switch in the
  `prompt` (e.g. "open in English, then switch to Hindi").

Show the planned personas in plain language and **approve before each `POST`**. Capture each
`personalityId`.

## Phase 3 — Create the scenarios (cover the real paths, not just the greeting)

A **scenario** is one simulated conversation with a **goal** the worker must achieve to "pass". Build a
set that exercises the **branches and failure paths**: happy path, missing/invalid data, the caller
who won't cooperate, guardrail/jailbreak attempts, **prompt-injection / social-engineering** (exercise
`manipulationDetectionEnabled`), **scope-drift over a long chat** (exercise `focusGuardrailEnabled`),
language switching, tool-failure handling. Three
ways to create them — pick per goal, usually (a) for breadth + (b) for the precise edge cases:

> ## ⚠ `aiAgentChannelType` is what makes a scenario runnable on the right channel — always set it
>
> All three creation paths take `aiAgentChannelType`, and it decides how `/simulation/run` executes the
> scenario. **Set it explicitly to match the worker's agent type — never leave it to chance:**
>
> | Worker / agent | `aiAgentChannelType` |
> |---|---|
> | voice (`VOICE_AGENT`) | `VOICE` |
> | chat (`CHAT_AGENT`) | `CHAT` |
> | **structured output (`STRUCTURED_OUTPUT`)** | **`CHAT`** |
>
> Valid values are only `VOICE` and `CHAT` (`GET /scenario/dropdown-config` → `channelType`); there is no
> separate structured-output channel — the schema states `aiAgentChannelType` *"Must be CHAT for
> CHAT_AGENT / STRUCTURED_OUTPUT and VOICE for VOICE_AGENT"*, and a live SO agent reports
> `aiAgentChannelType: "Chat"`. The created record echoes back `channelTypeLabel` (`"Chat"`/`"Voice"`) —
> which is the **Type** column in the Simulations → Scenarios UI, so read it back to confirm.
>
> **Chat and SO scenarios run as real simulations** — verified live 2026-08-03, a chat worker scored
> `passRate 100.0` and an SO worker returned `PASS` through `POST /simulation/run`. Do **not** fall back
> to `POST /aiworker/run` or a hand-made chat session for these channels; that is a live run
> (`callMode: COPILOT`), it shows under Observability's live filter, and it yields no pass-rate.

- **(a) AI-generate (breadth)** — `POST /scenario/generate` (`GenerateScenarioRequest`:
  `aiWorkerId, version, instructions, numScenarios, personalityIds, generationType, aiAgentChannelType,
  llm`). `instructions` steers the generator toward the use cases you care about. This is **async**:
  the result `body` carries only `scenarioGenerationId` — read it as `<generation-id>` and **poll** by
  repeatedly calling `commotion_request` `{ "method": "GET", "path":
  "/scenario?scenarioGenerationId=<generation-id>&aiWorkerId=<worker-id>" }` until the returned `body`
  array is populated (there is **no generation-progress endpoint**). Keep `numScenarios` ≤
  `maxScenarioGenerationLimit`. **Verified caveat:** generation needs a **deployed (live)** worker —
  against a never-deployed draft it returns a generation id but produces **zero** scenarios (and no
  error). If it comes back empty, fall back to (b). `CHAT` channel is accepted by the API even though
  the UI marks auto-gen voice-only — and chat/SO scenarios then simulate normally (verified live).
- **(b) Manual (precise edge cases)** — `POST /scenario` (`ScenarioRequest`: `name, aiWorkerId, version,
  intent, complexity, pathType, personalityId, situation, userScript, scenarioGoal, extraContext,
  aiAgentChannelType`). Use `complexity`/`pathType` codes from the dropdown-config. `userScript` is what
  the simulated caller says/shares; `scenarioGoal` is the pass criterion the evaluator checks.
- **(c) From a real call** — `POST /scenario/generate-from-conversation` (`conversationId, aiWorkerId,
  version, aiAgentChannelType`) turns a recorded interaction into a regression scenario. Review and
  complete the generated fields.
  **Find the call worth converting with `commotion_analyzer`** — Call Analyzer is the observability
  surface here, and this is the highest-value scenario source you have, because it is drawn from real
  behaviour rather than imagination:
  ```
  commotion_analyzer { "path": "/api/calls?workerId=<worker-id>&limit=20" }   # ⚠ query BOTH <id> and <id>_<version>
  commotion_analyzer { "path": "/api/reports/latest" }                        # the fleet's stop_reason / error-rate profile
  ```
  Pick the calls whose `stopReason` or transcript shows the behaviour you want to pin, read
  `?fields=transcript` to get the caller's turns, and **copy them verbatim** — STT artifacts and garbled
  words are frequently the trigger, so tidying them up produces a scenario that can't reproduce anything.
  ⚠ **Optional** — if `commotion_analyzer` is absent, returns `{ "status": 403 }` (no Call Analyzer
  access), or only a role-reduced view, say so once and author scenarios from the eval results /
  imagination instead; never block on it.
  Endpoint map: `commotion-debug/references/call-analyzer-api.md`. (`conversationId` is a BE-side id, not
  the Call Analyzer `callId` — bridge via `GET /conversation/worker-conversations?workerId=` matched on
  `voiceInteractionId`; if that doesn't resolve, author the scenario manually via (b) from the transcript.)

(Bulk CSV/Excel import also exists via `GET /scenario/import/csv|excel` → fill the file → `POST
/scenario/bulk` — see the reference; reserve it for large hand-authored sets.)

## Phase 4 — Review and write  ·  HUMAN INPUT REQUIRED

Summarize the planned test set in plain language — the personas, and a short table of scenarios (name,
path type, what it tests, its goal) — **not** a raw JSON dump. Get an explicit "yes", then create
(personalities first so scenarios can reference their `personalityId`). Show each write; surface any
backend error and check it against the references before retrying.

## Phase 5 — Confirm the test set

- `commotion_request` `{ "method": "GET", "path": "/scenario?aiWorkerId=<worker-id>" }` → read each
  record's `version` from `body` to know which version it belongs to.
- `commotion_request` `{ "method": "GET", "path": "/personality" }`.

Show the user the created scenarios (and their ids) for the version under test, and hand them to
`commotion-run-evals` (step 3) — that skill selects scenarios + runs-per-scenario and runs the
simulation. If the user wants to go straight to running, continue into the run-evals skill.

## Principles

- Ground before you draft; never invent a field that isn't in the schema (`commotion_schema`).
- A test set is only as good as its **coverage** — design personas + scenarios from the domain's real
  happy/failure/jailbreak paths, not a template. The `scenarioGoal` is the pass criterion, so make it
  concrete and checkable.
- Everything is scoped to `(aiWorkerId, version)` — create at the version under test; `version` is in
  each response, not a list filter.
- AI generation is **async** — poll `GET /scenario?scenarioGenerationId=` until the scenarios appear;
  respect `maxScenarioGenerationLimit` / `maxScenarioRunLimit` from the dropdown-config.
- Show every write before you make it (personalities before scenarios that reference them).
- If a platform call errors, `commotion_request` returns the backend's status + body — read it and
  check it against the references before retrying.
