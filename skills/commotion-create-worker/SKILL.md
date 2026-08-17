---
name: commotion-create-worker
description: >-
  Build and configure a Commotion voice or chat worker (AI agent) from a described goal, end to
  end — ground in the live config schema, interview for the goal, draft the worker (name, system
  prompt, voice + languages, guardrails), provision and enable its agent(s), configure Settings (pronunciation dictionaries,
  state variables) when the use case needs them, and deploy on approval.
  Use this whenever the user wants to create / build / set up a worker, voice agent, assistant, or
  bot for a use case — e.g. "make a voice agent that books dealership test drives in Hindi and
  English", "set up a multi-agent support bot for my client" — even if they don't say the word
  "worker". Handles the worker lifecycle (draft↔live versions, single vs multi-agent, enabling the
  agent, the voice/language schema). Calls the Commotion backend through the thin Commotion MCP server
  (OAuth — no API key in the transcript).
allowed-tools: Read, AskUserQuestion, Bash, Skill, mcp__commotion-dev3__commotion_request, mcp__commotion-tcuat__commotion_request, mcp__commotion-tcprod__commotion_request, mcp__commotion-dev3__commotion_schema, mcp__commotion-tcuat__commotion_schema, mcp__commotion-tcprod__commotion_schema, mcp__commotion-dev3__commotion_analyzer, mcp__commotion-tcuat__commotion_analyzer, mcp__commotion-tcprod__commotion_analyzer
---

# Commotion: Create a Worker

Turn a described goal ("a voice bot that books dealership test drives in Hindi and English") into a
configured, deployed Commotion worker. You supply the judgment — the name, the system prompt, the
voice/guardrail choices, the agent instructions — and you make the platform I/O through the connected
**Commotion MCP** server's two tools (`commotion_request` / `commotion_schema`). This skill carries
the endpoints, and you fetch request schemas live from the OpenAPI spec. Every write to the platform
is **shown to the user and approved before it happens**.

A worker is the **orchestrator** that holds and routes to its **agent(s)** — the actual
conversational behaviour lives in the agents. So creating a working worker is two things: configure
the worker (orchestration, voice, guardrails), then provision + **enable** its agent(s).

**Deploy is never automatic.** Going live is an explicit, user-gated step: always ask the user with
`AskUserQuestion` and get a clear "yes" before `deploy` (Phase 10). Drafting/creating/editing on a
draft is fine to do as you go (each write shown), but **never deploy a worker live without
confirmation.**

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

## When to use this

The user wants to create / build / set up a worker, voice agent, assistant, or bot. To *change* an
existing **live** worker, see "Editing a live worker" in `references/aiworker-lifecycle.md` (revert
to a draft → edit → redeploy) — the drafting and agent guidance below still applies.

## How this skill talks to the platform (read first)

All platform I/O goes through the connected **Commotion MCP** server — two tools, no scripts, no keys:

- **`commotion_request`** — one authenticated call: `{ "method": "GET|POST|PUT|DELETE", "path": "/…",
  "body": <JSON, for writes> }` → `{ "status", "body" }` (a non-2xx is **returned, not thrown** —
  read it and adjust). Pass a **path** (the base URL is fixed server-side) and the request payload as
  `body` — no temp files.
- **`commotion_schema`** — a bundled request schema: `{ "schema_name": "AiWorkerRequest" }` → the JSON
  Schema with its `$defs`. Any component name in the live spec works. **Never invent a field that
  isn't in the schema.**

**Auth is automatic (browser login).** The Commotion MCP handles auth via OAuth: on first use it
opens a Commotion login in the browser, then attaches the user's token to **every**
`commotion_request` / `commotion_schema` call for you — the raw token never enters the conversation.
Never ask the user for an email/password or a token, and don't pass a `token` argument. dev3
attributes the worker to the signed-in user (`createdByUserId` = their email). If the tools aren't
available at all, the MCP isn't connected — ask the user to add/authorize it via `/mcp` (see
`references/api-and-auth.md`).

**Read ids from results, not `jq`:** `commotion_request` returns the parsed `body` — after a create
the new id is `body.id`; from a list it's `body[0].id`. Feed that id into the next call's `path`.

The full endpoint map, error semantics, and schema-name list are in `references/api-and-auth.md` —
the single "how to call it" reference. Field *shapes* always come from `commotion_schema`; the
reference files are the *behavior* the schema doesn't tell you.

**Execution rules:** one phase at a time, in order; read the reference named by a phase before you
act on it; show every write before you make it. Phases 7, 8, 8.5 are **use-case-driven** — run each
when the goal calls for it (grounding; tools; remembered state; pronunciation), not by default and not
never. Judging what the worker actually needs — from the goal and the platform's use-case patterns — is
the point of the skill: add the feature that keeps the prompt clean, skip the one that would just be
noise. **Caveat on Phase 8.5:** its two halves are not equally optional — **state variables are
frequently *necessary*** for any worker that collects or looks up data (decide them in Phase 1 and
create them here), whereas the **pronunciation** half is genuinely usually *discovered later* from a
sim run. Don't let "8.5 is optional" cause you to skip the memory the use case needs.

## Phase 0 — Ground yourself in the real schema (always, before drafting)

Never invent field names or values. Read the contracts from the server first:

1. `commotion_schema` `{ "schema_name": "AiWorkerRequest" }` → the exact JSON Schema of the worker
   body (bundled with `$defs`). Cached for the session.
2. `commotion_request` `{ "method": "GET", "path": "/aiworker/metadata" }` → valid *values* and
   defaults (`agentSetupType` options; `guardrailConfig` toxicity categories/ranges + PII behaviours;
   `llmConfig` retry range). Top-level keys: `voiceConfig`, `guardrailConfig`, `workerConfig`, `llmConfig`.
3. `commotion_request` `{ "method": "GET", "path": "/aimodel?pageSize=200" }` → valid model / provider
   codes — for the voice block **and** for the primary + fallback models (Phase 3).
   ⚠ **`pageSize` is not optional.** `/aimodel` paginates at ~10 rows, and the bare call returns **10
   junk `azure_openai` test rows and zero Commotion models** (verified live 2026-08-06) — which reads
   falsely as "this workspace has no Commotion models, so I must pick a third party." The paged call
   shows `commotion-3.6-35b` (`isDefault: true`), `commotion-4-31b-it`, `commotion-3.6-27b`.

For the agent body fields (`AiAgentRequest`), see `references/agents-and-orchestration.md`; for
attaching source material / FAQ grounding, see `references/knowledge-and-rag.md`; for guardrails,
fallback models, and structured output, see `references/control-and-reliability.md`; for Settings —
pronunciation dictionaries and state variables (Phase 8.5) — see
`references/settings-variables-pronunciation.md` (their request schemas are `AiPronunciationDictRequest`
and `AiWorkerVariableSchemaRequest`); for skills — instruction blocks the agent loads on demand
(Phase 8.6) — see `references/skills-and-progressive-disclosure.md` (`AiWorkerSkillRequest`).

If the goal implies the worker must **act** (do something, not just answer), also ground in the tool
surface: `commotion_request` `GET /ai-worker-tool/metadata` (the built-in action catalog) and
`commotion_schema` for whatever kinds you'll attach — see `references/tools-and-capabilities.md`.

## Phase 1 — Understand the goal (interview only for what's missing)  ·  HUMAN INPUT

Extract: business goal, language(s), **voice or chat**, domain, tone, hard constraints, whether the
work is **one job (single agent)** or **several distinct skills that should be routed between
(multi-agent / workflow)**, and **which optional capabilities** the goal needs — knowledge grounding
(Phase 7), tools/actions (Phase 8), **whether the worker must remember caller-provided values across
the call (so it doesn't re-ask) or pre-load profile/account/booking data — i.e. state variables (Phase
8.5)**, structured output, guardrails beyond the safety floor. **Decide the state-variable question
here, not at the end** — most transactional use cases (booking, renewal, account lookup) need to
retain identifiers (PNR, passenger name, order id, dates) and that belongs in a variable, not the
prompt. Ask only for what you can't infer — `AskUserQuestion`, batched, few.

## Phase 2 — Choose the setup type

**Infer the setup type from the use case — the user will rarely say "multi-agent".** Read the goal
and decide whether the work splits into distinct, specialised responsibilities. If it does, prefer
**`MULTI_AGENT`** so each agent owns its specialty and does it well (a billing agent, a renewal
agent, an FAQ agent, …) instead of one giant prompt trying to do everything. Use one agent only when
the work genuinely is a single responsibility. **When you're unsure how to split it, ask the user**
(`AskUserQuestion`).

- **`MULTI_AGENT`** — specialist agents collaborate; the worker's `workerLevelPrompt` is the
  **orchestrator** that routes each request to the right agent. Each specialist is a separate,
  focused prompt.
- **`SINGLE_AGENT`** — one agent owns the whole job. Simplest, but one prompt carries everything.
- **`WORKFLOW`** — a fixed, predefined sequence of steps.

Tell the user which you chose and why. (Setup type is changeable later, but only while the worker is
a draft — see the lifecycle reference.)

Choose the setup type purely on the **use case** (above) — prompt visibility doesn't constrain it.
Both `POST` and `PUT` put a UI-visible prompt on an agent (verified live 2026-08-07), so the only
difference in Phase 6 is agent count: `SINGLE_AGENT` **`PUT`s the auto-provisioned default**;
`MULTI_AGENT` repurposes the default as the first specialist and **`POST`s** the rest. See
`references/agents-and-orchestration.md`.

**Structured output** is a `SINGLE_AGENT` variant: when the goal is to **return a strict, parseable
shape** for a downstream system to consume (not hold a conversation), set `structuredOutputEnabled:
true` and plan a single `STRUCTURED_OUTPUT` agent (Phase 6). Single-agent only.

## Phase 3 — Draft the worker config (this is the value you add)

Build a candidate `AiWorkerRequest` grounded in Phase 0. Hold it as the `body` you'll pass to
`commotion_request` in Phase 5.

- **`name`** — short, human, from the goal.
- **`agentSetupType`** — from Phase 2.
- **`workerGoal`** — one or two sentences: the outcome the worker drives toward.
- **`workerLevelPrompt`** — a **concise** worker-level role/identity + cardinal rules, NOT the full
  behaviour. **For `SINGLE_AGENT` the detailed system prompt / flow logic goes in the agent's
  `instructions` (Phase 6), not here** — verified live: on real single-agent workers
  `workerLevelPrompt` is a short role line (~100 chars) while the agent's `instructions` carry the
  actual behaviour. For `MULTI_AGENT`, this is the **orchestrator/routing** prompt (which agent
  handles what). Voice workers: spoken-style — short sentences, no markdown/lists/special characters,
  one question at a time, read names/numbers back (this style applies to whatever the agent *says*).
- **`greetingInstructions`** — how the worker opens the conversation. Write *instructions for the
  greeting*, not just a sentence ("Open with: … — do not add anything before it"), since the model
  still generates the turn. Only state/system variables may be referenced — **no `[tool:…]`,
  `[knowledge:…]`, `[agent:…]` or `[skill:…]`**; the backend does not reject them, it just leaves the
  raw token for the caller to hear. Agents carry their own `greetingInstructions`, so set it there
  instead when each specialist should open differently.
- **Voice + languages** (if voice-enabled) — set the voice block; list every language in
  `workerVoiceSettingsRequest.workerVoiceConfiguration.allowedLanguages` (that block also needs
  `model` / `provider` / `voiceId`, or let backend defaults stand). **Pick the `voiceId` from the voice
  catalogue: `GET /aiworkervoice?providerId=<tts provider>&modelId=<tts model>`** — it returns
  `voiceId` + `voiceName` + description + per-language sample audio URLs. It is **not** `/aimodel`
  (that's language models) and it is **not** UI-only; when the user says "choose a voice", list the
  candidates by name and let them pick. **Multilingual language rule
  (bake into the prompt):** stay in the caller's spoken language and continue the WHOLE call in it;
  treat a mobile number, policy number, OTP, or amount read out in English digits as normal data and
  do **NOT** switch the conversation language — and do **NOT** trigger the `Switch Language` built-in
  action — just because numbers are spoken in English. Only switch when the caller actually changes
  their conversational language. (Verified live: without this, the agent flips Hindi→English the
  moment the caller says their number in English.) Exact voice-block path in `references/aiworker-lifecycle.md`.
- **Guardrails** (`guardrailConfigRequest`) — **design them from the use case, don't apply a generic
  set.** **Direction matters — this is the mistake to avoid: `inbound` filters the *caller's* message
  before it reaches the model; `outbound` filters the *model's own output* before it's spoken/sent.**
  The model already has provider-side safety training, so it rarely emits toxic content on its own —
  **default to inbound-only for toxicity, and don't reflexively enable outbound.** Outbound is for the
  narrow cases where the *model's* text is the risk: masking PII it might echo back, blocking
  competitor/confidential terms in what it says, or a domain rule it could otherwise violate ("never
  quote a price / give financial advice"). If you can't name why the *output* is risky, leave outbound
  off. Think about what THIS domain handles and protect it, grounded in `/aiworker/metadata`:
  - Handles personal/financial data (insurance, banking, healthcare)? → **PII masking**
    (`builtInPiiCategoryConfigRequest` — Commotion's built-in detector) plus **regex masking**
    (`piiMaskingRegexPatternConfigList`) for the specific sensitive fields it sees (card numbers, account
    numbers, Aadhaar/SSN, policy numbers) with `MASK`/`REDACT` — this one legitimately runs **both**
    directions (mask what the caller says *and* what the model reads back). On **voice**, add
    `audioMaskingConfigRequest` to redact PII in call recordings.
  - Company/brand context? → **forbidden-word groups** (`forbiddenWordGroupsConfigRequest`) for
    competitor names, internal/confidential terms, off-limits topics — one group per intent, each with
    its own fallback message.
  - Any customer-facing bot → **toxicity `inbound`** (per-category on/off toggles in `toxicityCategories`,
    model `QWEN3_GUARD`) to catch abusive input; add **`outbound` toxicity only with a reason**. Add
    **custom checks** for domain rules (e.g. "never give medical/financial advice") — flip
    `inboundEnabled`/`outboundEnabled` and put each rule on the side that matters (a "don't accept X" rule
    is `inboundCustomGuardrailConfigs`; a "never *say* X" rule is `outboundCustomGuardrailConfigs`).
  - Exposed to adversarial users (public/customer-facing)? → **`advancedSafetyConfigRequest`**:
    `manipulationDetectionEnabled` is the first-class prompt-injection / jailbreak / social-engineering
    defense (it terminates or escalates) — prefer it over inbound toxicity alone for that threat. Turn on
    `focusGuardrailEnabled` for long/open-ended sessions that tend to drift off-scope.
  Pick the subset the use case warrants and justify each to the user — including **why each direction**.
  They apply in a fixed backend order — you don't set order; the only execution knob is `executionMode`
  (`STREAMING`/`BLOCKING`, voice-only). (e.g. a banking assistant → PII + card/account masking (both
  directions) + competitor forbidden-word group + **inbound** toxicity + manipulation detection + a "no
  financial advice" **outbound** custom check.)
- **Models + fallback** — choose the primary model and an ordered fallback so a provider hiccup
  doesn't take the worker down. **Where this lives depends on channel (verified live):** a **voice**
  worker sets its LLM provider/model **and its fallback** in the **Voice Settings** block
  (`workerVoiceSettingsRequest.workerLLMConfigurationRequest` + the voice-settings fallback fields —
  this is the "LLM Settings → Fallback Provider/Model" you see under Voice Settings in the UI), NOT in
  `workerAdvancedSettingsRequest` (which a voice worker / `VOICE_AGENT` rejects). A **chat** worker
  sets primary + `workerFallbackModelConfigurationRequestList` + `numberOfRetries` in
  `workerAdvancedSettingsRequest` — but that worker-level config is **not surfaced in the chat UI**
  (a chat worker has no worker-level Language Model panel), so if set only there the UI shows blank
  even though it drives the runtime. The UI's *Language Model* panel is on the **agent** (Agent →
  Advanced → Language Model), so **always set the model + fallback on the agent** for a chat worker —
  primary in `modelConfigurationRequestList`, fallback in
  `advancedSettingsRequest.languageModelSettingsRequest` — so they're always visible/editable in the UI
  (editing an agent needs the worker in DRAFT). Get codes from `/aimodel`. See
  `references/control-and-reliability.md`.
  **Provider rule — always `commotion`, and a Commotion provider needs no credential.** Pick a
  Commotion first-party provider for LLM **and** TTS **and** STT. A first-party provider's key comes
  from the platform environment, so its credential resolves automatically — **there is no credential
  for you to choose, and "Commotion" is never a credential to attach to someone else's provider.**
  Credentials are a **BYOK-only** concept (the schema calls `credentialId` *"Credential ID for BYOK
  (Bring Your Own Key) support"*) and exist on only two blocks: the fallback LLM and the
  simulator/eval-metric LLM. The third-party providers (openai/anthropic/cerebras/vayu/azure_openai,
  eleven-labs/cartesia/sarvam/smallest-ai) need a client credential that **no API field can carry** for
  STT/TTS/primary-LLM, and the backend accepts them anyway with `200` and no warning — leaving a
  saved-but-unrunnable provider+model whose Credential is blank or bound to the wrong thing. Verified
  live 2026-08-06: a `PUT` setting STT `provider: "eleven-labs"` returned `200` with the third-party
  provider round-tripped and no credential field anywhere in the response. So: **check
  `isThirdParty` in `GET /aiworker/metadata` before writing any provider**, and if the user wants an
  external one, tell them it is UI-only and leave the block on Commotion. Details + the field-by-field
  credential table: `references/aiworker-lifecycle.md` ("Providers and credentials").
- **Every agent needs its own primary model — `modelConfigurationRequestList`** (top-level on
  `AiAgentRequest`, `[{id, modelCode, providerCode}]` with all three keys). Applies to **voice and SO
  agents too**, not just chat: agents do **not** inherit the worker's model, and an agent created without
  this field lands with an empty model and a blank Language Model panel. Nesting it inside
  `languageModelSettingsRequest` is silently dropped. See `references/agents-and-orchestration.md`
  ("The agent's primary model").
- **Structured output** — if chosen in Phase 2, set `structuredOutputEnabled: true` here (the agent's
  schema is configured in Phase 6).

- **Anti-hallucination discipline (bake into the prompt).** If the agent's job depends on backend
  data (policy details, account status, eligibility, prices, "is X registered?"), the prompt MUST
  forbid stating or assuming any such fact unless a **tool/API actually returned it** in the
  conversation, and tell it what to do when it can't get the data (say it can't verify, hand off /
  call back — never guess). **Verified live:** a worker prompted to "call API 001 / API 002 …" with
  **no tools wired** confidently fabricated a backend result (declared a number "not registered" and
  proceeded) — a prompt that merely *names* an API does not make the agent call anything. Worse
  (verified live): when the prompt says "call API 001" with no registered tool, the model **fabricates
  a generic `api_call(...)` tool**, the platform returns `function 'api_call' is not registered`, and
  the agent **loops re-asking** for the same input. The real fix is to **register each API as a custom
  tool (Phase 8) and reference it by its action name** (`[tool:rmn-check-228]`) in the prompt, so the
  agent calls a real registered tool. Until tools are wired, the grounding rule keeps it honest. Keep
  the rule even after tools exist, for tool failures/empty results.
- **Don't re-ask for what's already given (bake into the prompt).** Once the caller HAS provided a
  detail, acknowledge it and move on — don't ask for the same thing again. (If the caller hasn't
  actually given it, or it was unclear/incomplete, asking — once — is correct.) Call each tool at most
  once per attempt; on a tool error/empty result, take the failure path ONCE (can't-verify →
  transfer/callback) rather than looping back to re-ask for information you already have.

Shapes, valid values, and worked examples for guardrails / fallback / structured output are in
`references/control-and-reliability.md`.

## Phase 4 — Show the draft and get approval  ·  HUMAN INPUT REQUIRED

Summarize in plain language (name, setup type, what it does, languages, guardrails, and the planned
agent(s)) — not a raw JSON dump. Get an explicit "yes" before any write.

## Phase 5 — Create the worker

Call `commotion_request` `{ "method": "POST", "path": "/aiworker", "body": <the AiWorkerRequest> }`,
then read the new worker id from the result: **`WORKER_ID = body.id`** (reused in every later call).

`POST /aiworker` returns a **DRAFT at version 0**. Capture the `id`. A new worker is provisioned with
a **default agent**, initially **disabled** — named "Chat Agent" on a chat worker and **"Voice Agent"
on a voice worker** (verified live; its `agentType` starts `null`). (A draft isn't visible to
`GET /aiworker/{id}`, which is live-only — confirm via `GET /aiworker` (list) if needed.)

**Worker names are globally unique (verified live 2026-07-10).** Creating with a name that already
exists returns `status 400` with `body` "*A worker with the name 'X' already exists*" — it is **not**
thrown, so read the status and react: pick a distinct name (confirm the new name with the user) or,
if the user means the existing worker, `GET /aiworker` (list), find it by name, and reuse its `id`
instead of creating a duplicate.

## Phase 6 — Provision + enable the agent(s)  ← the step people miss

Agents can only be created/edited while the worker is a **DRAFT**. **Golden rule (verified live
2026-08-07): `PUT /aiagent/{id}` writes the prompt and it renders in the UI editor** — into a blank
editor *and* over an existing prompt. So write prompts with `PUT` and keep the agent id stable; a
stable id is what stops scenarios (and anything else pinned to `aiAgentId`) from silently breaking.
List what's there first: `GET /aiagent?workerId=<worker-id>&version=0`.

- **`SINGLE_AGENT`** — `PUT` the auto-provisioned default; no delete, no re-POST:
  1. `commotion_request` `{ "method": "GET", "path": "/aiagent?workerId=<worker-id>&version=0" }` →
     the default agent id is `body[0].id` (grab its `modelConfigurationResponseList[0]` too).
  2. `commotion_request` `{ "method": "PUT", "path": "/aiagent/<default-id>", "body": <the agent,
     aiAgentEnabled:true> }` (body shape below).

  The default is born with the right `agentType` for the worker's channel (`CHAT_AGENT` /
  `VOICE_AGENT` / `STRUCTURED_OUTPUT`), so resend it unchanged — **`PUT` cannot change `agentType`**
  (`400 "Cannot change agent type…"`); that one case still needs delete + re-POST.
- **`MULTI_AGENT`** — `PUT` the default into your first specialist, then `POST /aiagent` each
  additional one with its own focused `instructions` + `aiAgentEnabled:true` (only `POST` can *add*
  agents). Disabling the default instead (`PUT` `aiAgentEnabled:false`) and POSTing every specialist is
  equally fine. The worker's `workerLevelPrompt` (Phase 3) is the orchestrator that routes to them.

Agent body (same for `PUT` and `POST`): `{aiWorkerId, version:0, name, description, agentType,
instructions, aiAgentEnabled:true, modelConfigurationRequestList}`. **`PUT` is a full replace — resend
every field you want to keep**, not just `instructions`. The request `agentType` echoes back as
`aiAgentType` (the `agentType` key reads back `null` — that's normal; the type still sticks).

Either way: **put the full prompt in the agent's `instructions`; keep `workerLevelPrompt` concise.** To
later revise it, `PUT` again — repeat `PUT`s overwrite the editor correctly, so there is no reason to
delete and re-POST for a prompt change. (POSTing a second agent into a `SINGLE_AGENT` worker still fails
`400 "Single Agent setup allows only one agent"`.)

**Structured-output agent (strict parseable shape).** ⚠ It needs its **own worker** — the flag is
mutually exclusive with voice and single-agent-only, so a `STRUCTURED_OUTPUT` agent cannot sit beside
other agents (verified live 2026-08-07: `POST /aiagent` on a voice `MULTI_AGENT` worker → `400
"Structured output is not enabled on the associated worker"`). If you set `structuredOutputEnabled: true`
(Phase 3), the default agent is auto-born as **`STRUCTURED_OUTPUT`** (disabled) — verified live. Follow
the **same single `PUT` as any single-agent worker**: `PUT /aiagent/{defaultId}` with
`{instructions:"…extract into the schema, no prose…", aiAgentEnabled:true,
structuredOutputConfig:{maxRetries, schemaFields:[…]}}` — the type is already right, so nothing needs
deleting. (`POST /aiagent` also accepts `STRUCTURED_OUTPUT` → 200, verified 2026-07-21, if you'd rather
create a fresh one.) The `schemaFields` shape (types, enums, nested objects, validation) is in
`references/control-and-reliability.md` / `references/agents-and-orchestration.md`.

**FAQ agent (answers strictly from docs).** When the goal is "answer questions from this material —
don't make things up," provision an **FAQ agent** — `agentType` **`FAQ_CHAT` or `FAQ_VOICE`** (the bare
`FAQ` value is in the enum but **not creatable**, verified live 2026-08-07; so is `CUSTOM`). Two
gotchas (verified live): FAQ types **must** be created with `POST /aiagent/standard`
(`POST /aiagent` rejects them — only VOICE_AGENT/CHAT_AGENT/STRUCTURED_OUTPUT), and the standard
agent is born **disabled with empty instructions** — follow up with `PUT /aiagent/{id}` to add
strict-grounding `instructions` (*answer only from the attached knowledge; if it isn't there, say
you don't know — never invent, no outside lookups*) and set `aiAgentEnabled: true`. FAQ instructions
are **`PUT`-only** (`POST /aiagent` rejects FAQ types and `/aiagent/standard` has no `instructions`
field) — which is fine now that `PUT` renders in the UI editor. An FAQ agent is
only useful once a knowledge base is attached and indexed (Phase 7). See
`references/agents-and-orchestration.md` for the full pattern.

If the API enable ever fails, fall back to enabling the agent in the Commotion UI, then continue.
See `references/agents-and-orchestration.md` for the agent fields, `agentType` values, and the rules.

## Phase 7 — Attach knowledge (optional — when the use-case needs grounding)

Skip this phase if the worker needs no source material. Otherwise attach a knowledge base so the
worker **grounds** its answers in it (grounding is automatic once knowledge is created and indexed
for the worker's `aiWorkerId` — there is no RAG toggle). Pick the source(s) the user has; the full
recipes (field shapes, enums, the presigned-PUT) are in `references/knowledge-and-rag.md`:

- **Inline / pasted text** → `POST /aiworker/file-upload/text` (`{content, fileName, fileType}`) →
  `POST /aiworker/knowledge/bulk` → `POST /aiworker/knowledge/index`.
- **Uploaded document** (PDF/docx/txt) → `POST /aiworker/file-upload/url` (`{fileName, fileType}`),
  then **PUT the file bytes to the returned `preSignedUrl` yourself** — `curl -X PUT --upload-file
  ./doc.pdf -H 'x-ms-blob-type: BlockBlob' "<preSignedUrl>"` (bytes go straight to Azure Blob
  Storage, **not** through the backend; the header is required or Azure 400s; success is `201`) →
  `POST /aiworker/knowledge/bulk` → `POST /aiworker/knowledge/index`.
- **Existing global KB** → `GET /aiworker/knowledge/global` →
  `POST /aiworker/knowledge/by-global/{globalId}?aiWorkerId=<worker-id>` (already published — no index step).
- **A website (crawl)** → a **different plane entirely**: `GET /aiworker/km-setting/<workerId>` → read
  `settingId` → `PUT /aiworker/km-setting/setting/<settingId>` with
  `{entityId, webCrawlerConfig: {enabled, seedUrls, crawlDepth, maxPages, syncFrequency, …}}`. That single
  PUT is the whole job — **no `file-upload`, no `bulk`, no `index`** — the crawler creates and indexes its
  own entries. But **it only runs on its schedule (no API trigger exists)**, so nothing appears until then.
  **You MUST report this back**: say that crawling is configured in the worker's settings, name the actual
  frequency (daily/weekly/monthly), and say the entries will be created and indexed on their own — an empty
  knowledge list until then is expected, not a failure. Otherwise the user is left staring at an empty KB
  wondering what broke. If they need the content now, snapshot the page as `TEXT_UPLOAD` instead.
- **One specific web page, needed now** → fetch and strip it yourself, then upload as
  `TEXT_UPLOAD` (the "Fetched web page" recipe) — a point-in-time snapshot, not a crawl. Say which it is.

Run `commotion_schema` `{ "schema_name": "CreateAiWorkerKnowledgeItemRequest" }` first if unsure of the bulk item shape.
Indexing is **synchronous** but the material isn't searchable instantly — **poll
`GET /aiworker/knowledge?aiWorkerId=<worker-id>` and wait until each item's `aiWorkerKnowledgeStatus`
is ready before deploying**. Show the user what you're attaching before each write.

**Then bind the KB to each grounded agent (required).** Worker-level attach alone does *not* make an
agent use it — the agent's prompt must reference the KB. Over the API this is a mention token in the
agent's `instructions`: `{..., instructions: "<prose telling it to search the knowledge base>\n\n
[knowledge:<knowledge name>|id:<knowledgeId>]"}` — compose the token into the prompt and write it the
**Phase-6 way** (`PUT /aiagent/{id}`, full body; it renders in the UI editor).
There is no separate agent↔knowledge field. See `references/knowledge-and-rag.md` ("Binding knowledge
to an agent").

## Phase 8 — Attach tools (think hard about what should be a tool)

Don't treat this as "skip unless the worker obviously acts." **Actively work out what the worker
should NOT be doing in the prompt and turn that into tools** — every lookup, status check, link
generation, record write, or external call belongs in a tool, so the prompt orchestrates and the
tools do the work (the prompt shouldn't carry data or fake results). **When you're unsure whether
something should be a tool, ask the user.**

- **Every API the flow calls MUST be a registered tool — never let the agent "call an API" from the
  prompt.** Naming an API in the prompt (e.g. "call API 001") does NOT make a call: the model
  fabricates a generic `api_call(...)`, the platform returns `function 'api_call' is not registered`,
  and the agent loops (verified live). Register each API as a `custom-tool` (`POST
  /ai-worker-tool/custom-tool`) and reference it by its **action name** (`[tool:rmn-check-228]`) in the
  agent's `instructions`. Read each tool's action name from
  `GET /ai-worker-tool?aiWorkerId=…&version=…` → `actionMetaDataOutputList[].actionName`.

The full per-kind recipes, body shapes, HITL, and the projection model are in
`references/tools-and-capabilities.md`. Tools attach to the **draft** worker (body carries
`aiWorkerId` + `version`), so do this before the deploy gate.

- **Decide what it must do**, and map each need to a kind: a platform built-in (end call, transfer) →
  `POST /ai-worker-tool/built-in-actions` (codes from `GET /ai-worker-tool/metadata`); an arbitrary
  HTTP API → `POST /ai-worker-tool/custom-tool` (an HTTP wrapper); custom in-process Python logic
  (transform, compute, format) → `POST /ai-worker-tool/code-block` (test the source first with
  `POST /ai-worker-tool/code-block/run`); an external MCP server → `POST /ai-worker-tool/mcp-server`;
  a managed SaaS app (Zoho, Slack, …) → a
  **connector**: `GET /ai-worker-tool/integration-apps` → `GET /ai-worker-tool/app-actions` /
  `GET /ai-worker-tool/webhooks` → `POST /ai-worker-tool/credential` (OAuth) →
  `POST /ai-worker-tool/connector` (see the connector recipe in the reference); another Commotion
  agent (A2A) → discover its card with `GET /.well-known/agent.json/{workerId}` and call it with
  `POST /a2a/{workerId}` (A2A is a separate resource, not an `ai-worker-tool` — see the reference's A2A note).
- **Worker vs agent (verified live).** A tool is *created* on the worker (`aiWorkerId` + `version`) —
  that's its only structural home; there is no agent↔tool field on the API. An **agent only calls a
  tool its prompt references**: embed a mention token in `instructions` — `[tool:<action name>]`
  (name only, no id; the action name comes from `GET /ai-worker-tool?aiWorkerId=…&version=…`'s
  `actionMetaDataOutputList[].actionName`, e.g. `lookup-order-189`). Same family as
  `[knowledge:<name>|id:<id>]`, `[agent:<name>|id:<id>]` (hand off to another agent), and `[var:…]`.
  So: create on the worker, then add `[tool:…]` to the agent's `instructions` and write it the **Phase-6
  way** (`PUT /aiagent/{id}`, full body; it renders in the UI editor) — that's
  how you scope a tool to a specific agent. See `references/tools-and-capabilities.md` ("Binding a tool
  to an agent").
- **Built-ins:** the catalog defaults (`end_call`, `switch_language`) are **already configured** on
  every worker — re-adding one is a 400. Add only non-defaults (e.g. `transfer_to_human`). Built-in
  actions have **no** `hitlMode`.
- **Code-block tools** run **sandboxed Python** (`language:"PYTHON"`) — good for logic on data you
  already have, but the sandbox has **no network/filesystem** (route external calls through a
  `custom-tool`). It supports `hitlMode`, `outputAsJson`, and `toolUsageTypes` (`LLM`/`PRE_LOAD`).
  Its `actionMetaDataOutputList` is empty — bind by `codeBlockMetadataOutput.lowerCaseName`
  (`[tool:<name>]`, used as-is). Test with `POST /ai-worker-tool/code-block/run` before attaching.
  Details in `references/tools-and-capabilities.md`.
- **HITL:** `hitlMode: "REQUIRE_APPROVAL"` lives on **connector, MCP-server, and code-block** actions
  (not built-in); at runtime that action pauses for approval and resumes via `POST /aiworker/continue`.
- **Connector credentials are validated** — a dummy/invalid key gives `200 {"id":"","success":false}`
  (no error). **Bind a connected `credentialMetaDataInput` IN THE CONNECTOR CREATE BODY — it is
  functionally required.** Attach a connector without one and it registers to nothing (empty
  action names/schemas + the UI banner *"Tools could not be registered — MCP … token is missing"*);
  there is **no PUT to add the credential afterwards** (update schema has no credential field), so it
  can only be fixed by delete + re-create. Also pass `appIconUrl`/`appTags` + per-action
  `actionDisplayName` so it renders like a UI connector. OAuth creds are minted in the UI; reuse a
  connected one from `GET /ai-worker-tool/credentials`. See `references/tools-and-capabilities.md`.
- **MCP-server tools** currently fail with a backend `500` on every create (verified live) — a dev3
  defect, not your input; don't promise this kind until BE fixes it.
- **Auto-capabilities (turn on, don't attach):** *reasoning* via
  `advancedSettingsRequest.languageModelSettingsRequest.reasoningEffortEnabled:true` +
  `reasoningEffort:LOW|MEDIUM|HIGH` (model must support it — see `/aimodel`). Not a tool. **State
  variables** are their own resource (configured in Phase 8.5, `/ai-worker-variable-schema`), also not
  a tool — an agent reads one in its prompt via `[var:<title>]`; a `LOADED` variable references a tool
  you created here.
- **Show every write before you make it** (especially each HITL gate); confirm with
  `GET /ai-worker-tool?aiWorkerId=<worker-id>&version=0`.

## Phase 8.5 — Settings (state variables + pronunciation — driven by the use case, not bolted on)

Two worker **Settings** subsystems, each its own worker-scoped resource created on the **draft** (body
carries the worker id + `version`), shown before each write. **These are not blanket-optional add-ons —
decide each from the goal** (the way you design guardrails from the use case in Phase 3): reach for the
one the use case calls for so the *prompt stays clean* instead of carrying that work itself. Full field
shapes and the verified-live gotchas are in `references/settings-variables-pronunciation.md`.

**State it out loud before moving on:** *does this worker need to remember anything the caller gives it,
or pre-load any record?* If yes — and for booking / renewal / account / order / lookup flows the answer
is almost always yes — that is a **state variable**, and skipping it pushes "remember X" into the prompt
where it bloats and misbehaves. This decision was flagged in Phase 1; resolve it here, don't silently
drop it.

- **State variables** (`/ai-worker-variable-schema`) — **add these whenever the goal needs the worker to
  remember caller-provided values (so it doesn't re-ask) or to pre-load profile/CRM/account data.** This
  is frequently *necessary*, not optional: a variable keeps "remember X / fetch X once" out of the prompt
  (which would otherwise bloat and misbehave). `POST /ai-worker-variable-schema {workerId, version,
  title, variableType, variableSource, availability, …}`. **`EXTRACTED`** = the LLM pulls it from the
  conversation (nothing else needed); **`LOADED`** = fetched from a tool and **requires `loadingStrategy`
  (`PRE_LOADING`/`DYNAMIC_LOADING`) + `toolReference`** (400 otherwise). The id comes back as **`id`**.
  **Creating a variable does not bind it** — the consuming agent must reference it in `instructions` as
  `[var:<title>]`, composed into the prompt and written per the Phase-6 rule (`PUT /aiagent/{id}` with
  the full body; it renders in the UI editor). A `LOADED` variable's `toolReference`
  points at a tool from Phase 8 (create that tool first); a code-block tool can also read a variable via
  its `stateVariables[]`.
- **Pronunciation dictionary** (`/ai-pronunciation-dict`) — **optional, and usually *discovered from a
  simulation run* rather than pre-guessed.** Add an entry when the worker speaks a brand name, acronym,
  SKU, or non-English term the TTS mangles — most reliably spotted when a sim transcript shows the
  tester-bot mis-hearing the term (see `commotion-run-evals` / `commotion-improve-worker`); don't try to
  enumerate them all up front. `POST /ai-pronunciation-dict {aiWorkerId, version, inputText,
  pronunciation, pronunciationType}` (`pronunciationType` ∈ `ALIAS`|`IPA`|`CMU`|`SYMBOL`|`PHONEME` —
  prefer `ALIAS`, e.g. `NPCL` → `"N-P-C-L"`). The id comes back as **`pronunciationDictId`** (not `id`);
  `inputText` is unique per worker+version. List with `GET /ai-pronunciation-dict?workerId=<id>&version=<v>`.

Both resources are edited by id (`PUT /ai-pronunciation-dict/{pronunciationDictId}` /
`PUT /ai-worker-variable-schema/{variableId}`) and bulk-deleted (array body). The same
use-case-then-simulation-signal discipline applies to **guardrails / forbidden words** (Phase 3 /
`control-and-reliability.md`): design them from the domain, then tune from what sim runs reveal (an
off-limits answer → add a guardrail; a blocked legitimate request → loosen one). "Agent re-asks info it
already has" → state variable; "bot mispronounces our brand" → pronunciation entry — see
`commotion-improve-worker`.

## Phase 8.6 — Skills (keep the long instructions out of the context window)

A **skill** is a named, described block of instructions stored beside the worker. The agent's prompt
carries only a manifest of every attached skill's `name` + `description`; the body arrives **only when
the agent calls the built-in `read_skill(name)` tool** on a turn that needs it. Verified live: a
cancellation question cost 870 input tokens on the manifest turn and 1053 after the body loaded — the
policy text costs nothing on the turns that don't touch it. Full behaviour:
`references/skills-and-progressive-disclosure.md`.

**Ask yourself before Phase 9: is any part of this prompt a self-contained topic the agent needs only
sometimes?** If yes, it belongs in a skill. The three cases that earn one:

- **The prompt has grown long** — the classic symptom is the agent ignoring rules buried mid-prompt.
  Split each topic out; what's left is routing.
- **Per-topic policy blocks** — cancellation rules, eligibility criteria, fee tables. Only the topic in
  play is ever in context, so one policy can't bleed into another.
- **A procedure several agents share** — one skill, bound to each agent, one place to edit.

Not a skill: retrieval over documents (that's knowledge, Phase 7) or anything with a side effect or a
live lookup (that's a tool, Phase 8). `read_skill` only returns text you wrote.

Create on the draft, then bind:

```
POST /ai-worker-skill  {aiWorkerId, version, name, description, instructions}   # all five required
   → the id comes back as `aiWorkerSkillId` (not `id`); `name` is unique per worker version
PUT  /aiagent/{id}     {…full body…, "aiWorkerSkillIds": ["<id>", …]}           # THIS is the binding
```

Two rules that decide whether skills work at all:

- **Write the `description` as a routing signal, not a title.** It is the *only* thing the model sees
  when deciding to load the skill — say when to use it, in the caller's words ("Use whenever the caller
  asks to cancel, reschedule, or asks about charges for missing an appointment"). A vague description
  is the number-one reason a skill silently never fires.
- **`aiWorkerSkillIds` binds; `[skill:<name>]` only hints.** Unlike `[tool:…]`, the token is never
  expanded — it stays literal in the prompt. Write it *and* bind the id. A `[skill:X]` with no matching
  id is a dangling reference: no error, the agent just answers from general knowledge instead.

⚠ Skill ids are **not** checked for ownership — an id from another worker is accepted with a `200` and
silently never resolves. Only bind ids from `GET /ai-worker-skill?aiWorkerId=<this>&version=<this>`.

## Phase 8.7 — After Call Work (only if the call should trigger post-call work)

If the use case needs something to happen *after* the call — a CRM update, a wrap-up summary, a
follow-up task — that is **ACW**: a second worker the platform fires when the call ends. Ask whether
the goal implies post-call work; if it doesn't, skip this phase.

Build order is fixed, because the reference is validated:

1. `POST /aiworker` a non-voice worker named **`[ACW] <worker name>`** (the convention).
2. `PUT` its auto-provisioned agent with the wrap-up instructions and `aiAgentEnabled: true`.
3. **Deploy it** — `POST /aiworker/<acw-id>/deploy?version=0`. A draft target fails with
   `400 "ACW live worker not found with ID: …"`.
4. Point the *voice* worker at it with `PUT /aiworker/{id}`, inside the voice block:
   `workerVoiceSettingsRequest.workerAdvancedVoiceSettingsRequest.acwMetadataRequest =
   {aiWorkerId, aiAgentId, includeSessionState, transcriptFormat: "JSON"|"MARKDOWN"}`.

⚠ Two traps, both verified live: **the agent-level `acwMetadataRequest` is unreachable on a voice
agent** (`400 "Advanced settings is not supported for VOICE_AGENT type."`) — use the worker block; and
**don't send `voiceAgentType`** alongside it unless you're also wiring telephony, or you get
`400 "connectionId cannot be blank when voiceAgentType is given."` Remember `PUT /aiworker/{id}` is a
full replace — resend the whole voice block. Details: `references/aiworker-lifecycle.md`.

## Phase 9 — Deploy readiness gate

Confirm with `GET /aiagent?workerId=<worker-id>&version=0` before deploying:

- `SINGLE_AGENT` → **exactly one enabled agent** (else deploy 400s "requires exactly one enabled
  agent, but found 0").
- `MULTI_AGENT` → the agents the orchestrator needs are present and enabled.
- If you attached knowledge (Phase 7) → every item's `aiWorkerKnowledgeStatus` is ready (not still
  indexing/failed), so the worker actually grounds on it from the first live conversation.
  **Exception — a crawler-configured worker legitimately has zero items** until its first scheduled
  crawl, so an empty knowledge list is *not* a readiness failure here. Instead verify
  `GET /aiworker/km-setting/<worker-id>` shows `status: "SUCCESS"` and `webCrawlerConfig.enabled: true`,
  and **tell the user the worker will have no grounding until the first crawl runs** — that's a real
  deploy consideration, not a footnote.
- If you attached tools (Phase 8) → `GET /ai-worker-tool?aiWorkerId=<worker-id>&version=0` shows them as expected.
- If you created skills (Phase 8.6) → check **both directions**: every `[skill:<name>]` in an agent's
  `instructions` has a matching id in that agent's `aiWorkerSkillIds`, **and** every id in
  `aiWorkerSkillIds` is referenced by a `[skill:<name>]` somewhere in those `instructions`. Also
  confirm every id came from *this* worker's `GET /ai-worker-skill`. None of these mismatches produces
  an error, so this check is the only thing that catches them.
- If you wired ACW (Phase 8.7) → the referenced ACW worker is **LIVE** and its agent is enabled.

## Phase 10 — Deploy  ·  ALWAYS ASK FIRST

**Never deploy without an explicit user "yes".** Once readiness passes, summarise what will go live
and ask with `AskUserQuestion` (e.g. "Deploy this worker live now?" — Deploy now / Keep as draft).
Only on a clear yes, call `commotion_request`
`{ "method": "POST", "path": "/aiworker/<worker-id>/deploy?version=0" }`.

A fresh worker's first deploy is **version 0**. If the user is not ready, leave it as a draft (you
can persist a draft without going live with `commotion_request` `POST /aiworker/<worker-id>/draft?version=0`). Deploying
is the one irreversible-feeling step for the user — gating it on confirmation is mandatory, not optional.

## Phase 11 — Confirm live

Call `commotion_request` `{ "method": "GET", "path": "/aiworker/<worker-id>" }`. This now returns the
live worker — show the user the result and its agents.

## Phase 12 — Prove it behaves (don't stop at CRUD)  ·  HUMAN INPUT

Creating the worker is not the goal — a worker that *behaves well* is. **A build that is never tested
is not done**, and for anything beyond a throwaway demo this phase is mandatory, not a footnote.

**Recommend the quality loop by default.** After confirming live, and *especially for a production /
non-trivial use case*, don't just note that testing exists — **offer to run it**. `AskUserQuestion`:
"The worker is built and live. For a production use case I recommend running the quality loop next —
generate test scenarios, evaluate the pass-rate, and iterate until it clears a threshold. Run it now?"
(Run the quality loop / Just a quick spot-check / Stop here.)

- **On "run it":** invoke the **`commotion-quality-loop`** skill (via the `Skill` tool), handing it the
  worker id you just built — it owns build→scenarios→evals→improve and the deployed-voice prerequisite
  (it will convert/deploy as needed). This is the right path for a production statement; stopping at a
  built worker leaves it unverified.
- **On "quick spot-check":** drive a few real turns yourself (below).
- **Prerequisite for the loop (state it):** automated simulations/evals run on **voice, chat AND
  structured-output** workers (one endpoint, `POST /simulation/run`; the channel comes from each
  scenario's `aiAgentChannelType`) — what they do need is the worker **deployed at least once**, since a
  never-deployed worker has no runtime. So the Phase 10 deploy is what makes the loop runnable,
  whatever the channel.

Text spot-check (the lightweight path):

Call `commotion_request` `{ "method": "POST", "path": "/aiworker/run", "body": {"workerId":
"<worker-id>", "messageText": "<a realistic opening line>", "conversationId": "t1", "sessionId":
"t1", "userId": "t1"} }`, then continue the conversation by reusing the same `conversationId`/`sessionId`.

`POST /aiworker/run` runs the worker in text and returns `{response,status,...}` (parse tolerantly —
the body can contain raw newlines; the endpoint is occasionally flaky, so retry on 5xx). Pick
scenarios that exercise the **branches and the failure paths**, not just the happy greeting. Watch
for the agent **asserting backend facts it never fetched** (the #1 failure) — if it invents
data, tighten the grounding rule (Phase 3) and/or wire the tools (Phase 8), redeploy, and re-test.

**Then read the spot-check back in Call Analyzer** — the reply text alone hides the thing you most need
to see. Each `/aiworker/run` session appears as a `COPILOT` session:

```
commotion_analyzer { "path": "/api/chat/sessions?workerId=<worker-id>&limit=5" }   # find the session
commotion_analyzer { "path": "/api/chat/session/<session-id>" }                     # runs, errors[], perf
```

The payoff: **`toolCallMetrics` / `registered_tools` prove whether the tools you wired in Phase 8
actually fired**, and a `[tool:<name>]` in your prompt with no matching entry in `registered_tools` is a
**dangling reference** — at runtime it returns `"Error: function '<name>' is not registered."` and the
agent will often fabricate an answer rather than admit the failure. That is exactly the #1 failure above,
and it is invisible in the reply text but obvious here. Endpoint map:
`commotion-debug/references/call-analyzer-api.md`. (For a **voice** worker use the voice surface instead:
`/api/calls?workerId=<worker-id>` then `/api/call/<id>?fields=transcript,metrics` — and query both the
bare `<worker-id>` and `<worker-id>_<version>` forms.) ⚠ `commotion_analyzer` is optional — skip this if
it isn't connected, if it returns `{ "status": 403 }` (your user has no Call Analyzer access), or if
your role only returns a partial view.
Editing the live worker means revert-to-draft → edit the agent at the new draft version → redeploy
(see `references/aiworker-lifecycle.md`).

## Principles

- Ground before you draft; never invent a field that isn't in the schema (`commotion_schema`).
- A worker isn't usable until its agent is **enabled** — treat Phase 6 as mandatory, not optional.
- **Always a Commotion provider for LLM / STT / TTS — and then there is no credential to pick.**
  "Provider" and "credential" are not two choices: a **first-party Commotion provider takes its key
  from the platform environment, so its credential is resolved automatically and is not yours to set**.
  A credential only exists for **BYOK** — the schema says so (`credentialId`: *"Credential ID for BYOK
  (Bring Your Own Key) support"*), and it appears on **only** the fallback-LLM and simulator-LLM blocks.
  So never pair a third-party provider with a Commotion credential, and never write a third-party
  provider into the STT/TTS/primary-LLM blocks at all: they have **no credential field**, the backend
  accepts them with `200` and no warning, and the UI then renders a provider+model with a wrong or
  blank Credential that cannot run. If the user wants an external STT/TTS/LLM, it's UI-only — say so and
  change nothing. Details: `references/aiworker-lifecycle.md` ("Providers and credentials").
- **Every agent carries its own `modelConfigurationRequestList`** (top-level, `{id, modelCode,
  providerCode}` from `/aimodel`) — chat, voice and SO alike. Agents inherit **nothing** from the worker's
  model choice, and mis-nesting this field inside `languageModelSettingsRequest` is dropped silently.
  Read `modelConfigurationResponseList` back to confirm.
- **Voices come from `GET /aiworkervoice`, not `/aimodel`.**
- **The web crawler is configured on `km-setting` (`webCrawlerConfig`), never on a knowledge item.** That
  one PUT is the whole job — the crawler creates and indexes its own entries. Never hand-create a
  `WEBSITE_CRAWL` item; it always fails. There's **no manual trigger over the API**, so a crawl only
  lands on its schedule (`DAILY` ≈ midnight). **This is the one knowledge source where finishing looks
  like nothing happened, so always tell the user what you arranged** — crawling is set up in the worker's
  settings, it runs daily/weekly/monthly (name which), and the entries create and index themselves; an
  empty KB until the first run is expected. Snapshot the page as `TEXT_UPLOAD` if they need content
  immediately. See `references/knowledge-and-rag.md`.
- **Never send a partial `chunkingConfig` on the `km-setting` PUT** — it permanently and unrecoverably
  destroys the worker's KB index. PUT only the fields you're changing.
- **After each write, read the response back** rather than trusting the `200`. Several fields on this API
  are silently dropped when mis-shaped (agent model config) or silently accepted-but-broken
  (non-Commotion providers, a hand-made `WEBSITE_CRAWL` item, a partial `chunkingConfig`).
- Grounding needs both halves: knowledge must be **created and indexed** — attaching without
  indexing (or deploying before indexing finishes) means the worker has nothing to ground on.
- Show every write before you make it; the user approves going live.
- Tools are created on the **worker** (draft only); an agent uses one by **naming it in its prompt**
  (like knowledge), so scope per agent there. Flag risky connector/MCP actions `REQUIRE_APPROVAL`.
- Guardrails + fallback models are worker-definition config (set on the draft, shown before write);
  guardrail order is backend-enforced. Structured output is **single-agent only** (`structuredOutputEnabled`
  + a `STRUCTURED_OUTPUT` agent).
- Settings (pronunciation dictionaries, state variables) are **their own worker-scoped resources**
  created on the draft, not fields on `AiWorkerRequest`. A state variable isn't used until an agent
  names it in its prompt (`[var:<title>]`); a `LOADED` variable needs a `loadingStrategy` + a tool.
- Agents are editable only on a draft; editing a live worker means reverting it to a draft first.
- If a platform call errors, the helper surfaces the backend's status + message — read it and check
  it against the reference notes before retrying.
