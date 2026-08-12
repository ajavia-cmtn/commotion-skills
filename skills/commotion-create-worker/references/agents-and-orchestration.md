# Agents, single vs multi-agent, and orchestration

A **worker** is a container; its conversational behaviour lives in one or more **agents** (the
`/aiagent` resource, reached through the two Commotion MCP tools `commotion_request` /
`commotion_schema` — see `api-and-auth.md`). This file is the agent-side companion to
`aiworker-lifecycle.md`.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/aiagent?workerId=&version=&pageNumber=0&pageSize=10&sortDirection=DESC` | list (find the default agent, verify enablement) |
| GET | `/aiagent/{id}?version=N` | one agent's full record |
| POST | `/aiagent` | create an agent on a DRAFT worker (`AiAgentRequest`) |
| POST | `/aiagent/standard` | create a *standard* agent, e.g. FAQ (`CreateStandardAgentRequest`) |
| PUT | `/aiagent/{id}` | update an agent in place — `instructions`, `name`, `aiAgentEnabled`, model config. **The prompt renders in the UI editor** (verified live 2026-08-07); keeps the agent id (`AiAgentRequest`) |
| DELETE | `/aiagent/{id}?version=N` | delete an agent (the `version` query param is **required**) |

Body shapes: `commotion_schema` `{ "schema_name": "AiAgentRequest" }` / `{ "schema_name":
"CreateStandardAgentRequest" }`.

## The golden rules (verified against dev3)

1. **Agents can be created/edited only while the worker is a DRAFT.** Creating one on a live worker
   fails: `400 "Agent can only be created when worker is in draft status. Use a draft worker."` To
   change a live worker's agents, revert it to a draft first (see the lifecycle reference).
2. **A new worker is auto-provisioned with one default agent, DISABLED.** Deploy requires it enabled.
3. **`aiAgentEnabled: true` is the deploy gate.** The error `"requires exactly one enabled agent, but
   found 0"` means the agent exists but is disabled — enable it with `PUT /aiagent/{id}`.
4. **`SINGLE_AGENT` allows exactly one agent — total.** Trying to add a second fails: `400 "Cannot
   create another agent. Single Agent setup allows only one agent."` To have more than one, switch
   the worker to `MULTI_AGENT` first (`PUT /aiworker/{id}` on the draft), *then* `POST /aiagent`.
5. **Agent type is immutable via `PUT` (verified live).** Changing an existing agent's `agentType`
   (e.g. `CHAT_AGENT`→`VOICE_AGENT`, as when a worker is switched to voice) fails: `400 "Cannot change
   agent type. Please delete the existing agent and create a new one…"`. **Delete the agent and
   re-POST** it with the new type (`DELETE /aiagent/{id}?version=N` → `POST /aiagent`). An
   instructions-only `PUT` (same type) is fine.

## Setup types

| `agentSetupType` | Meaning | Agents | Worker prompt role |
|---|---|---|---|
| `SINGLE_AGENT` | One agent does everything | exactly 1 | the agent's behaviour |
| `MULTI_AGENT` | Specialists collaborate | many | **orchestrator** — routes each request to the right agent |
| `WORKFLOW` | Predefined sequence of steps | as the flow needs | the flow definition |

Switching type is allowed **only on a draft** (verified: SINGLE_AGENT → MULTI_AGENT via
`PUT /aiworker/{id}`, then a second agent attached and deployed).

**Model config in a mixed multi-agent worker resolves per agent type (verified live).** A voice-enabled
`MULTI_AGENT` worker can hold both a `VOICE_AGENT` and a `CHAT_AGENT`, and there is no single shared
"worker model": the `VOICE_AGENT` draws its LLM from the worker's Voice-Settings block (it can't hold
its own — `advancedSettingsRequest` on a `VOICE_AGENT` is rejected), while each `CHAT_AGENT` member
carries its own agent-level model config. See `control-and-reliability.md` ("Mixed multi-agent worker").

**Where the prompt lives (verified live).** The **detailed system prompt / flow logic goes in the
agent's `instructions`**, not `workerLevelPrompt`. `workerLevelPrompt` is a short worker-level role
line (~100 chars, or the orchestrator/router for MULTI_AGENT); the agent's `instructions` carry the
full behaviour. The runtime composes the two.

**`PUT /aiagent/{id}` writes the prompt AND it renders in the UI editor (verified live 2026-08-07 —
this reverses the old delete-and-re-POST rule).** Both write paths now put a UI-visible prompt on the
agent, so **prefer `PUT` for anything prompt-related** — it keeps the agent id, and a stable id is what
keeps scenarios, ACW references and anything else pinned to `aiAgentId` from silently breaking.

The live test that settled it (dev3, both workers confirmed in the UI by a human):
- **Chat worker, PUT-only default agent** — worker `PUT UI Test 20260807`, its auto-provisioned "Chat
  Agent" never deleted or re-POSTed, just `PUT /aiagent/{defaultId}` with `instructions`. The prompt
  **rendered** in the editor. So a `PUT` into a *blank* editor populates it.
- **Voice worker, POST-created agent, then edited by PUT** — worker `POST UI Test Voice 20260807`:
  default deleted → `POST /aiagent` with a prompt (rendered) → then `PUT /aiagent/{id}` with different
  text. The editor **picked up the new text**. So a `PUT` also *overwrites* an existing prompt; it
  doesn't just fill a blank one and it doesn't leave the editor stale.
- **Every creatable `agentType`, same day** — workers `All Agent Types PUT Test 20260807` (voice,
  MULTI_AGENT: `VOICE_AGENT`, `CHAT_AGENT`, `FAQ_CHAT`, `FAQ_VOICE`) + `SO Agent PUT Test 20260807`
  (`STRUCTURED_OUTPUT`). All five were written `HI` + a `[tool:…]` token, then `PUT` to `Hello` +
  a `[knowledge:…]` token, then — after a deploy — `PUT` again to `hey aarya`. Every state rendered.
  So **`PUT` visibility is not type-specific**, and it survives swapping one mention-token family for
  another. The `FAQ_*` types are included: they can *only* be written by `PUT`, and they render.
- **A `PUT` through a deploy cycle** — both workers deployed, reverted with
  `POST /aiworker/{id}/draft?version=0`, `PUT` at v1, redeployed. Rendered at v1.

What this changes:
- **Prompt create or revise → `PUT /aiagent/{id}`.** No delete, no re-POST, id unchanged.
- **`SINGLE_AGENT` → just `PUT` the auto-provisioned default.** Its `agentType` is already correct for
  the worker's channel (`CHAT_AGENT` / `VOICE_AGENT` / `STRUCTURED_OUTPUT`), so one `PUT` carrying
  `{name, description, instructions, aiAgentEnabled:true, modelConfigurationRequestList}` finishes the
  agent. Delete-then-POST still works and is not wrong — it's just extra calls plus a new id for no gain.
  ⚠ **`POST /aiagent` on a `SINGLE_AGENT` worker is refused outright** (verified live 2026-08-10):
  `400 "Agent creation failed: Cannot create another agent. Single Agent setup allows only one agent."`
  The default is not merely the convenient path here — it is the only one, short of deleting it first.
- **`MULTI_AGENT` → `PUT` the default into your first specialist, then `POST` the rest.** A worker needs
  more than one agent here, and only `POST` can add agents. (Disabling or deleting the default instead is
  still fine.)
- **Delete + re-POST is now only for what a `PUT` genuinely can't do:** changing `agentType` (immutable
  via `PUT` — rule 5 above), or removing an agent you no longer want. Those *do* mint a new `aiAgentId`
  — re-point anything referencing the old one (see `commotion-debug/references/repro-and-gates.md`).
- **A blank editor box is still not proof of a missing prompt** — `GET /aiagent/{id}` (`instructions`)
  + a `POST /aiworker/run` test remain the source of truth.
- **The worker must be a DRAFT — a live one rejects the `PUT`** (verified live 2026-08-07):
  `400 "Agent update failed: Agent can only be updated/deleted when worker is in draft status. Use a
  draft worker."` To edit a deployed worker: `POST /aiworker/{id}/draft?version=<live>` → `PUT` the
  agents **at the new draft version** → `POST /aiworker/{id}/deploy?version=<draft>`. The draft revert
  **preserves every agent id** (see `aiworker-lifecycle.md`), so this whole cycle is id-stable too.

`POST /aiagent` with `instructions` in the create body is unchanged and still renders — use it when you
are genuinely *adding* an agent. When freeing the single-agent slot to add a differently-typed agent,
`DELETE /aiagent/{defaultId}?version=0` first (the `version` query param is **required** — without it:
`400 "version is required"`); a direct POST while the default exists fails `400 "Single Agent setup
allows only one agent"`.

**Voice worker default agent (verified live).** A voice worker's auto-provisioned default agent is
named **"Voice Agent"** (chat workers get "Chat Agent") and starts disabled. The request field
`agentType` is echoed in the **response** as **`aiAgentType`** — so after a `POST`/`PUT` with
`agentType:"VOICE_AGENT"`, read it back from `aiAgentType` (it sticks); the `agentType` key itself is
request-only and reads back `null`, which is *not* a failure to save. `aiAgentEnabled: true` is the
deploy gate.

**Prompt set via API drives the runtime, and (since 2026-08-07) renders in the UI editor.** The
agent's `instructions` set over the API — by `POST` **or** `PUT` — are what the worker actually runs on
(confirmed by `POST /aiworker/run`, the agent followed the prompt) and they now show in the UI's rich
prompt editor. If an editor box ever *does* come up empty, that still isn't proof the agent has no
prompt: check `GET /aiagent/{id}` (`instructions`) and a test run before rewriting anything.

**Anti-hallucination (verified live).** An agent whose prompt says "call API X" but has **no tool
wired** will *fabricate* the result rather than call anything — e.g. it declared a phone number "not
registered" with nothing backing it. Naming an API in the prompt does not make the agent call it.
Worse (verified live): when the prompt says "call API 001" with no registered tool, the model
**fabricates a generic `api_call(...)` tool**, the platform returns `function 'api_call' is not
registered`, and the agent then **loops — re-asking for the same input** instead of failing. Two
fixes, use both:
1. **Wire each API as a real tool** and reference it by its **action name** (`[tool:rmn-check-228]`)
   in the agent's `instructions` (see `tools-and-capabilities.md`) — so the agent calls a registered
   tool, not a hallucinated `api_call`.
2. **Give the prompt an explicit grounding rule** (keep it even after tools exist, for tool
   failures/empty results): never state/assume a backend fact unless a tool actually returned it; if
   you can't get it, say you can't verify and hand off / call back — never guess.

**Don't re-ask for what's already given (verified live — bake into the prompt).** Once the caller HAS
provided a detail, acknowledge it and move on — don't ask for the same thing again. (If they haven't
actually given it, or it was unclear/incomplete, asking *once* is correct.) Call each tool **at most
once per attempt**; on a tool error/empty result take the failure path **once** (can't-verify →
transfer/callback) rather than looping back to re-ask for information you already have. This is the
same loop the fabricated-`api_call` case triggers — the grounding rule and the call-once rule
together keep the agent from spinning.

## The agent body (`AiAgentRequest`)

Required: **`aiWorkerId`**, **`version`**, **`name`**, **`description`**. Useful optional fields:

- **`agentType`** — the enum holds seven values, but **only five can actually be created** (verified
  live 2026-08-07 by attempting all seven on one worker):

  | `agentType` | How to create it |
  |---|---|
  | `VOICE_AGENT`, `CHAT_AGENT`, `STRUCTURED_OUTPUT` | `POST /aiagent` (or `PUT` the auto-default) |
  | `FAQ_CHAT`, `FAQ_VOICE` | `POST /aiagent/standard`, then `PUT` for `instructions` |
  | **`FAQ`**, **`CUSTOM`** | **no path — both endpoints reject them** |

  `POST /aiagent` with `FAQ` or `CUSTOM` → `400 "Agent creation failed: Only VOICE_AGENT, CHAT_AGENT &
  STRUCTURED_OUTPUT agent type is supported."` `POST /aiagent/standard` with either → `400 "Standard
  agent creation failed: Only FAQ_CHAT & FAQ_VOICE agent type is supported."` So `FAQ` and `CUSTOM` are
  enum values with no door into either endpoint — **don't offer them to a user or plan around them.**
  (Use `FAQ_CHAT`/`FAQ_VOICE` for the FAQ behaviour; there is no generic-channel `FAQ`.)
- **`instructions`** — the agent's system prompt / behaviour.
- **`aiAgentEnabled`** — boolean; must be `true` to count toward the deploy gate.
- **`greetingInstructions`** — how the agent opens the conversation. See "The first message" below.
- **`aiWorkerSkillIds`** — array of `aiWorkerSkillId`s to advertise to this agent. This is the real
  binding for skills; see `skills-and-progressive-disclosure.md`.
- Plus `aiAgentSubscriptionRequestList`, `aiAgentTriggerInputList`, `structuredOutputConfig`, `imageUrl`.

### The first message — `greetingInstructions`

A plain string on the agent (and, at worker level, on `AiWorkerRequest`) that governs the agent's
opening line. It is *instructions for the greeting*, not the literal text — "Open with: Northside
Dental, this is Riya — how can I help you today? Do not add anything before it." works better than a
bare sentence, because the model still generates the turn.

**The schema's rule: only state and system variables may be referenced — no tool, knowledge, agent or
skill references.** That makes sense (there is nothing to call yet at greeting time), but ⚠ **it is not
enforced** — a greeting containing `[skill:…]` and `[knowledge:…|id:…]` was stored with a `200`
(verified live 2026-08-10). Nothing rejects it and nothing resolves it, so the caller hears the raw
token. Treat it as your rule to keep, not the server's.

Survives deploy and revert-to-draft unchanged. It does **not** appear in the `/aiworker/run` text
system prompt — it is a session-start field for the delivered voice/chat channel, so verify it there
(the resolved voice context exposes it as `greeting_instructions`; see
`commotion-debug/references/call-analyzer-api.md`).

### ⚠ The agent's primary model: top-level `modelConfigurationRequestList` — EVERY agent, EVERY channel

**Every agent you POST or PUT must carry `modelConfigurationRequestList`, or it ends up with no model at
all** — `modelConfigurationResponseList: []`, a blank *Agent → Advanced → Language Model* panel
(Provider/Credential/Model all empty) and an agent the user cannot run. This applies to **chat, voice and
structured-output alike**; it is not a chat-only concern.

It is a **top-level array on `AiAgentRequest`**, and all three keys are required:

```jsonc
"modelConfigurationRequestList": [
  { "id": "69fc2c6ece21b786c1e3625f",     // the AiModel record id from GET /aimodel — REQUIRED
    "providerCode": "commotion",
    "modelCode": "commotion-3.6-35b" }
],
"advancedSettingsRequest": {               // sibling — tokens/temp/retries/fallback ONLY
  "languageModelSettingsRequest": {
    "maximumOutputTokens": 1024,           // REQUIRED — omitting it 400s the create
    "numberOfRetries": 1,
    "reasoningEffortEnabled": false,
    "temperature": 0.2 }
}
```

**The trap (verified live 2026-08-03).** `LanguageModelSettingsRequest` holds **no primary-model field** —
only `maximumOutputTokens`, `temperature`, `numberOfRetries`, `reasoningEffort*` and
`fallbackModelConfigurationRequestList`. Nesting the model inside it (e.g.
`languageModelSettingsRequest.languageModelConfigurationRequest`, mirroring the *worker* shape) is
**silently dropped**: the call returns `200`, the sibling settings round-trip, and only
`modelConfigurationResponseList: []` reveals the loss. Always read that array back after writing.

Note the deliberate asymmetry with the worker: at **worker** level the model *is* nested
(`workerAdvancedSettingsRequest.workerLanguageModelSettingsRequest.workerLanguageModelConfigurationRequest`,
which does round-trip). Agent ≠ worker here. Get `id`/`modelCode`/`providerCode` from `GET /aimodel`, and
keep the provider **`commotion`** (see `aiworker-lifecycle.md`, "Providers and credentials").

**⚠ `advancedSettingsRequest` is rejected outright for `VOICE_AGENT` (verified live 2026-08-10):**
`400 "Agent update failed: Advanced settings is not supported for VOICE_AGENT type."` The block is
chat/structured-output only. Two consequences: a voice agent takes its token/temperature/fallback
settings from the **worker**, and the `acwMetadataRequest` that `AdvancedSettingsRequest` advertises is
**unreachable on a voice agent** — wire ACW at worker level instead (`aiworker-lifecycle.md`).

**⚠ `temperature` is required whenever you do send the block (verified live 2026-08-10):** omitting it
→ `400 "languageModelSettingsRequest.temperature is required."`, even though the schema does not list
it under `required`. Send `maximumOutputTokens`, `temperature`, `numberOfRetries` and
`reasoningEffortEnabled` together.

### Auto-provisioned agents get the platform default — they do NOT inherit the worker's model

Verified live 2026-08-03, in both directions:

| Worker | Worker-level LLM set to | Default agent was born with |
|---|---|---|
| chat, `SINGLE_AGENT` | `commotion-3.6-27b` | `commotion-3.6-35b` |
| voice, `SINGLE_AGENT` | `commotion-3.6-27b` (voice LLM) | `commotion-3.6-35b` |

So **neither chat nor voice agents inherit the worker's model choice** — every auto-provisioned agent
(Chat / Voice / Structured Output) is born with the platform-default model `commotion` /
`commotion-3.6-35b` (the `isDefault: true` entry in `/aimodel`), plus `maximumOutputTokens: 1024`,
`temperature: 0.1`, `numberOfRetries: 0`, and **disabled**. If you want the worker's model actually used
by the agent, set it on the agent explicitly.

The **delete-default-then-POST** path is where the model goes missing: the auto-provisioned agent had a
default, and your replacement inherits nothing. Whenever you delete-and-re-POST, re-send
`modelConfigurationRequestList` along with the prompt. `PUT`ing the default instead sidesteps this — but
`PUT` is a full replace, so **resend `modelConfigurationRequestList` there too** or you wipe the model
the default came with.

`version` is the worker version you're editing (e.g. `0` for a fresh worker, or the draft's version
when editing a live worker's draft).

## FAQ agents (answer strictly from docs)

An **FAQ agent** answers only from attached knowledge — no invention, no live lookups. There is no
"strict" flag; the behaviour is **prompt-led**. Creating one has a sharp edge (verified live):

1. **FAQ types must be created via `POST /aiagent/standard`, NOT `POST /aiagent`.** The plain
   `POST /aiagent` rejects them: `400 "Only VOICE_AGENT, CHAT_AGENT & STRUCTURED_OUTPUT agent type is
   supported."` Use `POST /aiagent/standard` with `{agentType:"FAQ_CHAT", aiWorkerId:<id>, version:<draft>}`
   — or `FAQ_VOICE`. **`FAQ` itself is not creatable** (`400 "Only FAQ_CHAT & FAQ_VOICE agent type is
   supported."`, verified live 2026-08-07); pick the one matching the channel.
2. **A standard FAQ agent is born DISABLED with empty instructions.** Follow up with
   `PUT /aiagent/{id}` with `{... , instructions:"<strict grounding>", aiAgentEnabled:true}` to add
   the prompt and enable it. Strict-grounding instructions, e.g. *"Answer only from the worker's
   attached knowledge; if a topic isn't in it, say you don't know — never guess, no outside knowledge,
   no live lookups."*
3. **FAQ instructions are `PUT`-only — and that is no longer a visibility problem.** `POST /aiagent`
   rejects FAQ types (`400 "Only VOICE_AGENT, CHAT_AGENT & STRUCTURED_OUTPUT …"`) and
   `CreateStandardAgentRequest` (the `/aiagent/standard` body) has **no `instructions` field**, so the
   only way to set an FAQ prompt is the follow-up `PUT` in step 2. **Verified live 2026-08-07 directly
   on both `FAQ_CHAT` and `FAQ_VOICE` agents**: the `PUT`-written prompt renders in the UI editor, and
   re-`PUT`ing it updates the editor. The old "FAQ prompts are invisible in the UI" caveat is retired
   outright — FAQ is no longer an exception to anything.

An FAQ agent is only useful once a knowledge base is **attached and indexed** — see
`references/knowledge-and-rag.md`. `CHAT_AGENT`/`VOICE_AGENT`/`STRUCTURED_OUTPUT` agents are created
normally with `POST /aiagent` (instructions inline).

## Structured-output agents (the 4th type — strict parseable output)

A `STRUCTURED_OUTPUT` agent returns a **strict, schema-conforming shape** instead of free prose — for
when a downstream system parses the worker's output. It is **single-agent only** (one agent, total),
and the worker carries `structuredOutputEnabled: true`.

**⚠ It cannot share a worker with other agents (verified live 2026-08-07).** `structuredOutputEnabled`
is *"mutually exclusive with voice. Supported for Single Agent and Workflow setups"* (schema), so a
`STRUCTURED_OUTPUT` agent can't sit beside a `VOICE_AGENT` or in a `MULTI_AGENT` worker at all —
`POST /aiagent` on such a worker returns `400 "Agent creation failed: Structured output is not enabled
on the associated worker. Enable structured output on the worker first."` and the flag can't be enabled
there. **A structured-output agent needs its own worker.** The config + verified flow:

1. **Create the worker with `structuredOutputEnabled: true`** (`SINGLE_AGENT`). Verified live: the
   auto-provisioned default agent is then born as **`STRUCTURED_OUTPUT`** (disabled).
2. **`PUT` the default with the prompt + `structuredOutputConfig`** — same as any single-agent worker
   (the default is already `STRUCTURED_OUTPUT`, so no type change is needed and the id stays put):
   `PUT /aiagent/{defaultId} { name, description, agentType:"STRUCTURED_OUTPUT", instructions:"…extract
   into the schema, no prose…", aiAgentEnabled:true, structuredOutputConfig:{…} }`. The prompt renders in
   the UI editor (the 2026-08-07 PUT finding). Delete-then-POST also works and is still verified
   (2026-07-21: `POST /aiagent` accepts `STRUCTURED_OUTPUT` with `instructions` + `structuredOutputConfig`
   → `200`) — it just costs an extra call and a new `aiAgentId`.

`structuredOutputConfig` = `{ maxRetries, schemaFields:[ SchemaField… ] }`. Each **`SchemaField`**:
- `name`, `type`: `STRING | INTEGER | FLOAT | BOOLEAN | OBJECT`, `description`.
- `required`, `multiValue` (array of that type).
- `enumEnabled: true` + `enumValues:[…]` for a closed set.
- `properties` — nested `SchemaField`s when `type: OBJECT`.
- `validation` — `{ minLength, maxLength, pattern, format: DATE_TIME|TIME|DATE|EMAIL|DURATION,
  minValue, maxValue, multipleOf, minItems, maxItems, … }`.

```
DELETE /aiagent/{defaultAgentId}?version=<draft>          # free the single-agent slot
POST /aiagent  body:
{ aiWorkerId:<id>, version:<draft>, name:"Order Extractor",
  description:"Extracts an order summary", agentType:"STRUCTURED_OUTPUT", aiAgentEnabled:true,
  instructions:"Extract the order details into the schema; never add prose.",
  structuredOutputConfig:{ maxRetries:2, schemaFields:[
    { name:"order_id", type:"STRING", required:true, description:"The order id" },
    { name:"amount",   type:"FLOAT" },
    { name:"status",   type:"STRING", required:true, enumEnabled:true,
      enumValues:["PENDING","SHIPPED","DELIVERED"] } ] } }
```

Verified live: the schema round-trips intact, and `POST /aiagent` with `STRUCTURED_OUTPUT` +
`structuredOutputConfig` returns `200` (worker `6a5f153e…54e5`, 2026-07-21). That the agent
actually *returns* a conforming shape is a runtime behaviour — needs a live conversation to confirm.
See `references/control-and-reliability.md` for the worker-side `structuredOutputEnabled` + guardrails
+ fallback config.

## Recipes

**Enable a single-agent worker (the common case) — one `PUT` on the default:**
```
GET /aiagent?workerId=<id>&version=0                # the default agent, disabled, empty prompt
PUT /aiagent/<defaultAgentId>  { aiWorkerId:<id>, version:0, name, description,
                                 agentType:"VOICE_AGENT",          # already its type — resend, don't change it
                                 instructions:"<full prompt + [tool:]/[knowledge:]/[var:] tokens>",
                                 aiAgentEnabled:true,
                                 modelConfigurationRequestList:[{id, modelCode, providerCode}] }
# prompt renders in the UI + exactly one enabled agent → POST /aiworker/<id>/deploy?version=0
```
`PUT` is a **full replace** — resend every field you want to keep (name, description, agentType, model
config, triggers), not just `instructions`. Only reach for `DELETE` + `POST /aiagent` when you actually
need a *different* `agentType` than the default was born with; that mints a new `aiAgentId`.

**Build a multi-agent worker:**
```
# worker created/updated with agentSetupType = MULTI_AGENT (on a draft)
PUT  /aiagent/<defaultAgentId>  { ...full body..., name:"Billing", instructions:"...",
                                  aiAgentEnabled:true }            # repurpose the default as specialist #1
POST /aiagent  { aiWorkerId:<id>, version:0, name:"Renewals", description:"...",
                 agentType:"VOICE_AGENT", instructions:"...", aiAgentEnabled:true }   # POST each additional one
# the worker's workerLevelPrompt is the orchestrator that routes to these specialists
# → POST /aiworker/<id>/deploy?version=0
```
(Disabling the default with `PUT { ...keep fields..., aiAgentEnabled:false }` and POSTing every
specialist is equally valid — repurposing it just saves one agent slot.)
