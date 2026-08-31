# Tools & capabilities

How to give a worker the ability to **act**, not just talk — over the dev3 `ai-worker-tool` plane,
reached through the two Commotion MCP tools (`commotion_request` / `commotion_schema` — see
`api-and-auth.md`). A *tool* is a named capability attached to a worker; the agent(s) decide when to
call it during a conversation. This is the companion to `agents-and-orchestration.md` (what the agents
are) and `knowledge-and-rag.md` (what they know).

> Scope of this doc: the kinds wired today — **built-in actions**, **custom (HTTP API) tools**,
> **code-block tools** (sandboxed Python), **MCP-server tools**, the **connector** ecosystem
> (integration apps + OAuth credentials + webhooks), and **A2A** (calling another agent) — plus the
> cross-cutting model (worker-level projection, HITL).

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/ai-worker-tool?aiWorkerId=&version=&aiWorkerToolId=&searchText=&pageNumber=&pageSize=&sortDirection=` | list a worker's tools (or one tool by `aiWorkerToolId`) |
| DELETE | `/ai-worker-tool` | detach tools (array of ids in body; draft only) |
| GET | `/ai-worker-tool/metadata` | built-in action catalog (read first) |
| POST / PUT | `/ai-worker-tool/custom-tool[/{id}]` | HTTP-wrapper tool |
| POST / PUT | `/ai-worker-tool/built-in-actions[/{id}]` | built-in actions |
| POST / PUT | `/ai-worker-tool/code-block[/{id}]` | sandboxed Python tool |
| POST | `/ai-worker-tool/code-block/run` | test-run source in the sandbox (stateless) |
| POST / PUT | `/ai-worker-tool/mcp-server[/{id}]` | external MCP server (⚠ create 500s) |
| POST / PUT | `/ai-worker-tool/connector[/{id}]` | SaaS connector (credential + icon/tags settable on POST only; PUT is actions/webhooks) |
| POST | `/ai-worker-tool/credential` | store a credential |
| GET | `/ai-worker-tool/credentials?appIdentifiers=…` | list stored credentials |
| DELETE | `/ai-worker-tool/credential` | delete credentials (body `{"credentialIds":[…]}`) |
| GET | `/ai-worker-tool/integration-apps?identifiers=&pageNumber=&pageSize=` | available SaaS apps |
| GET | `/ai-worker-tool/app-actions?aiWorkerId=&version=&appIdentifier=&searchText=&…` | an app's actions |
| GET | `/ai-worker-tool/webhooks?appIdentifier=&searchText=&…` | an app's webhooks |
| POST / PUT | `/ai-worker-tool/a2a-agent[/{id}]` | attach another agent as an A2A tool |
| GET | `/.well-known/agent.json/{workerId}` · POST `/a2a/{workerId}` | A2A card / send |
| POST | `/aiworker/continue` | resume a run paused on a `REQUIRE_APPROVAL` action |

Body shapes: `commotion_schema` `{ "schema_name": "<Name>" }` — `CreateCustomToolRequest`,
`CreateBuiltInActionsToolRequest`, `CreateCodeBlockToolRequest`, `RunCodeBlockRequest`,
`CreateMcpServerRequest`/`UpdateMcpServerRequest`, `CreateConnectorToolRequest`/`UpdateConnectorToolRequest`,
`CreateA2aAgentToolRequest`/`UpdateA2aAgentToolRequest`, `CreateCredentialRequest`.

## The golden rules (verified against the dev3 spec)

1. **A tool is created on the worker; an agent uses it via its prompt (verified live).** Every
   create body carries only `aiWorkerId` + `version` — there is **no agent↔tool field** anywhere on
   the API (`AiAgentRequest`/`AiAgentResponse` have no tools field; the tool record has no agent
   field). So the tool's only structural home is the worker. An **agent uses a tool by naming it in
   its `instructions`** — the `/Actions` mention, the exact same pattern as `/Knowledge` — which is
   how you scope a tool to a *specific* agent. (Exception: a `WORKFLOW` worker binds a tool to a step
   by `aiWorkerToolId`.) So "attach to the agent" = create on the worker, then reference it in that
   agent's prompt. Don't say "tools are worker-wide" — that's only true until you write the prompts.
2. **Attach only on a DRAFT.** Like agents and knowledge, tools can only be created/updated while the
   worker is a draft. Attach during provisioning (after the agents exist, before deploy); editing a
   live worker means reverting it to a draft first (only one draft exists per worker at a time).
3. **Ground in the catalog + schema first.** `GET /ai-worker-tool/metadata` lists the built-in action
   catalog (the valid `builtInActionId` codes). `commotion_schema` `{ "schema_name": "<kind-schema>" }`
   returns the live JSON Schema for any kind's body — never invent a field.
4. **HTTP vs. code — two different kinds (verified live).** `custom-tool` wraps an arbitrary HTTP API
   (url + method + headers + query + body). For in-process logic (transform, compute, format, validate)
   there is now a separate **`code-block`** kind that runs **sandboxed Python** — but the sandbox has
   **no network and no filesystem**, so anything that reaches out must still be a `custom-tool`. Pick
   HTTP for external calls, code-block for logic on data you already have.
5. **Every API the flow calls MUST be a registered tool — naming it in the prompt does NOT call it
   (verified live).** A prompt that says "call API 001" with no registered tool makes the model
   **fabricate a generic `api_call(...)`**; the platform returns `function 'api_call' is not
   registered`, and the agent **loops, re-asking for the same input**. The fix: register each API as a
   `custom-tool` (`POST /ai-worker-tool/custom-tool`) and reference it by its **action name**
   (`[tool:rmn-check-228]`) in the agent's `instructions` — read the action name from
   `GET /ai-worker-tool?aiWorkerId=…&version=…` → `actionMetaDataOutputList[].actionName`. Keep the
   prompt's anti-hallucination/grounding rule (see `agents-and-orchestration.md`) even after the tool
   exists, for tool failures and empty results.
6. **HITL is in-band — on connector, MCP-server, and code-block actions (verified live).** Set `hitlMode`
   (`AUTO_RUN` | `REQUIRE_APPROVAL` | `ASK_FOR_DETAILS`) on a connector/MCP action, or in a code-block's
   `codeBlockMetadata`, in the create body; **built-in actions have no `hitlMode`** (a custom tool's
   auto-generated action exposes the slot but it isn't an input on create). A paused run resumes with
   `POST /aiworker/continue` (see below).
7. **Show every write before you make it.** Same rule as the rest of the skill — summarise the tool
   you're about to attach (and especially any `REQUIRE_APPROVAL` action) and get a yes.
8. **A connector needs its credential IN THE CREATE BODY, or it silently fails to register (verified
   live 2026-07-24).** Put a connected `credentialMetaDataInput.credentialIdentifier` on the
   `POST /ai-worker-tool/connector` call. Skip it and the tool attaches (`200`) but never registers —
   empty `actionName`/schemas and the UI banner *"Tools could not be registered — MCP … token is
   missing."* There is **no PUT to add a credential later** (the update schema has no credential field),
   so a bare connector can only be fixed by delete + re-create. Also pass `appIconUrl` + `appTags`
   (from `integration-apps`) and per-action `actionDisplayName` (from `app-actions`) in the same body so
   it renders like a UI-built connector (icon, Category, action chips) — these are POST-only too. Never
   attach a connector before a connected credential exists.

## Picking the kind

| The worker needs to… | Kind | Endpoint |
|----------------------|------|----------|
| End a call, transfer, or another platform built-in | built-in action | `POST /ai-worker-tool/built-in-actions` |
| Call an arbitrary HTTP/REST API you have a URL for | custom (HTTP wrapper) | `POST /ai-worker-tool/custom-tool` |
| Run custom Python logic (transform, compute, format, validate) in a sandbox | code block | `POST /ai-worker-tool/code-block` |
| Use actions exposed by an external MCP server | MCP-server | `POST /ai-worker-tool/mcp-server` |
| Use a SaaS app (Zoho, Slack, …) with managed auth | connector | `POST /ai-worker-tool/connector` |
| Call another Commotion agent | A2A | `GET /.well-known/agent.json/{id}` + `POST /a2a/{id}` |

## The bodies (run `commotion_schema` `{ "schema_name": "<Name>" }` for the live shape)

**Built-in actions** (`CreateBuiltInActionsToolRequest`) — `aiWorkerId`, `version`, and
`builtInActionMetaDataRequestList[]`; each entry's `builtInActionId` is a `code` from
`GET /ai-worker-tool/metadata` (`end_call`, `transfer_to_human`, `code_interpreter`,
`switch_language`), plus `description`, `builtInActionDisplayName`, and per-action params. **No
`hitlMode`.** The defaults (`end_call`, `switch_language`) are already on every worker — re-adding one
returns `400 "already configured"`, so only add the non-defaults.

**Custom HTTP tool** (`CreateCustomToolRequest`) — `aiWorkerId`, `version`, and `customToolMetadata`:
`name`, `description`, `url`, `customToolMethod` (`GET`/`POST`/`PUT`/`PATCH`/`DELETE`),
`headers` (object), `queryParams[]` and `body[]` (each field `{fieldName, fieldType:
STRING|NUMBER|BOOLEAN|JSON|OBJECT|ANY, required, defaultValue}`). The backend auto-generates the
action name/identifier from `name` (e.g. `lookup_order` → action `lookup-order-189`).

**Code-block tool** (`CreateCodeBlockToolRequest`) — `aiWorkerId`, `version`, and `codeBlockMetadata`.
Required in the metadata: `name` (unique within the version), `description`, `language` (**`PYTHON`**
is the only value today), `sourceCode`. Optional: `hitlMode` (this kind **does** support it, unlike
custom/built-in), `contextPrompt` (shown to the model for this tool), `outputAsJson` (parse stdout as
JSON — default `false`), `toolUsageTypes[]` (`LLM` = model decides when to call, `PRE_LOAD` = runs
before the conversation to load context; default `[LLM]`), and the variable refs `stateVariables[]`
(`{id, placeholder?}` — omit `placeholder` for a whole *extracted* variable, set it for a *loaded* leaf
path), `llmVariableIds[]`, `systemVariableIds[]`. Inside `sourceCode`, reference those via the
placeholder forms `{{[statevar:..]}}`, `{{[llmvar:..]}}`, `{{[sysvar:..]}}` (a distinct `{{[...]}}`
syntax — **not** the prompt-binding `[var:..]` token). The `id` in each `stateVariables[]` entry is a
state variable's **`id`** returned by `POST /ai-worker-variable-schema` — **create the variable in
Phase 8.5 first** (see `settings-variables-pronunciation.md`); in the platform UI this same reference is
the **`@variable`** insert (e.g. `@citizen_id`). So one state variable is consumable two ways: an agent
reads it via the prompt token `[var:<title>]`, and code reads it via a code-block `stateVariables[]`
entry. **The placeholder is injected as a pre-quoted literal** (verified live): write
`x = {{[statevar:title]}}` — the sandbox produces `x = "refund request"` — **not** `x = "{{[statevar:title]}}"`
(the extra quotes yield a `SyntaxError`). **Unlike custom tools there is no numeric-suffix
action name:** the tool's `actionMetaDataOutputList` is **empty** and the bindable name is
`codeBlockMetadataOutput.name`/`lowerCaseName` (used as-is, e.g. `format_policy_number`). **The sandbox
has no network and no filesystem** — for external calls use a `custom-tool`. Test the source before
attaching with `POST /ai-worker-tool/code-block/run` (`RunCodeBlockRequest`: `sourceCode` + optional
sample `stateVariableValues[]`/`llmVariableValues{}`/`systemVariableValues{}`/`outputAsJson`) →
`{success, stdout, stderr, errorMessage, executionId, durationMs}`.

**MCP-server tool** (`CreateMcpServerRequest`) — `mcpServerUrl`, `name`, `aiWorkerId`, `version`,
`requestTimeout`, `maximumTotalTimeout`, and `mcpServerHeaderRequest` (the headers used to
authenticate to that external server). The backend handshakes the server at create time to discover
its actions. **Known issue (verified live):** dev3 returns a generic `500 "Failed to create MCP
server"` on *every* attempt — unreachable public servers **and** a reachable, known-good streamable-HTTP
server (with and without auth). A bad body is a 400, so this is a **backend-side failure in the
create/handshake flow**, not an input/reachability problem — raise with BE before relying on MCP-server tools.

**Connector tool** (`CreateConnectorToolRequest`) — required: `aiWorkerId`, `version`,
`appMetaDataInput` (`appIdentifier` + display fields, from `GET /ai-worker-tool/integration-apps`),
`actionMetaDataListInput[]` (each `actionIdentifier` from `GET /ai-worker-tool/app-actions`, with a
per-action `hitlMode`). **⚠ `credentialMetaDataInput` (a `credentialIdentifier`) is functionally
REQUIRED — bind it in the create body (verified live 2026-07-24).** Without it the connector attaches
(`200`) but its backing managed MCP server **never registers**: the tool comes back with empty
`actionName`/`inputSchema`/`outputSchema` and no `credentialMetaDataOutput`, and the UI shows the red
banner *"Tools could not be registered — MCP server failed to register, or the MCP authentication token
is missing."* **You CANNOT fix this later with a PUT:** `UpdateConnectorToolRequest` has **no**
credential field (and no `appMetaData` field) — it carries only `actionMetaDataListInput[]` +
`toolWebhookMetaDataInputList[]`. So a credential-less (or wrong-icon/tags) connector must be **deleted
and re-created**. Grab a connected credential from `GET /ai-worker-tool/credentials?appIdentifiers=…`
(reuse one with `"connected": true`) or complete OAuth in the UI first.

**Populate the display metadata too, or the tool renders bare in the UI (verified live 2026-07-24).**
A minimal create (only `appIdentifier` + `actionIdentifier` + `hitlMode`) registers fine but shows a
generic "G" icon, an empty **Category**, and empty **Actions & Webhooks** chips — unlike a UI-created
connector. Thread these through from the discovery endpoints so it matches the UI:
- `appMetaDataInput.appIconUrl` ← `GET /integration-apps` → `iconUrl` (the app logo)
- `appMetaDataInput.appTags` ← `GET /integration-apps` → `integrationAppTags` (drives the **Category** column)
- `actionMetaDataListInput[].actionDisplayName` ← `GET /app-actions` → `displayName` (the action chips)

(`actionDescription` is accepted but **not persisted** — the backend derives descriptions from the
action schema at runtime, so don't rely on it.) Optional `toolWebhookMetaDataInputList[]` (from
`GET /ai-worker-tool/webhooks`). The update body (`UpdateConnectorToolRequest`) is partial — the
action/webhook lists you pass **replace** the existing ones. An app-action object is
`{identifier, displayName, existsInAiWorker}`.

**Credential** (`CreateCredentialRequest`, only `appIdentifier` required) — `name`, `displayName`,
`appIdentifier`, `authIdentifier`, and the auth payload. An app advertises its auth methods in
`GET /ai-worker-tool/integration-apps` → `credentials[]`: `authIdentifier` is `OAUTH_AUTHORIZATION_CODE`,
`OAUTH_CLIENT_CREDENTIALS`, or `OTHER` (API-key/token), each with an `inputConfigList` of fields.
- OAuth apps → pass `authorizationCode` (`code` + PKCE `codeVerifier`) from a completed consent flow.
- `OTHER` apps → pass the key(s) as a JSON string in `data` (e.g. `data:"{\"apiKey\":\"…\"}"`).
**The backend validates the credential** — an invalid/dummy key returns **`200 {"id":"","success":false}`**
(no error text), so you can't fake a working credential (verified with a dummy Clockify key). Credentials
are shared — one can back several connector tools.

## Recipes

**Built-in action** (no `hitlMode`; skip the catalog defaults `end_call`/`switch_language`):
```
GET  /ai-worker-tool/metadata                 # codes: end_call, transfer_to_human, code_interpreter, switch_language
POST /ai-worker-tool/built-in-actions  { aiWorkerId:<id>, version:<draft>,
  builtInActionMetaDataRequestList:[
    { builtInActionId:"transfer_to_human", builtInActionDisplayName:"Transfer to human",
      description:"Escalate to a human agent" } ] }
```

**Custom HTTP tool:**
```
commotion_schema { "schema_name": "CreateCustomToolRequest" }   # confirm the field shape
POST /ai-worker-tool/custom-tool  { aiWorkerId:<id>, version:<draft>, customToolMetadata:{
  name:"lookup_order", description:"Fetch an order by id",
  url:"https://api.example.com/orders", customToolMethod:"GET",
  headers:{ Authorization:"Bearer …" },
  queryParams:[{ fieldName:"orderId", fieldType:"STRING", required:true, defaultValue:"" }] } }
```

**Code-block tool** (sandboxed Python — test the source first, then attach):
```
commotion_schema { "schema_name": "CreateCodeBlockToolRequest" }   # confirm the metadata shape
# 1. test-run the source in the sandbox (stateless — no worker needed):
POST /ai-worker-tool/code-block/run  { sourceCode:"raw='123456789012'\nresult='-'.join([raw[i:i+4] for i in range(0,len(raw),4)])\nprint(result)" }
#   -> { success:true, stdout:"1234-5678-9012", executionId:"…", durationMs:… }
# 2. attach on the draft (multi-line source is just a string in the body arg — no temp file):
POST /ai-worker-tool/code-block  { aiWorkerId:<id>, version:<draft>, codeBlockMetadata:{
  name:"format_policy_number", description:"Format a raw policy number as XXXX-XXXX-XXXX",
  language:"PYTHON", sourceCode:"…" } }        # optional: hitlMode, contextPrompt, outputAsJson, toolUsageTypes
# 3. read the bindable name back — it's NOT in actionMetaDataOutputList (empty for CODE_BLOCK):
GET /ai-worker-tool?aiWorkerId=<id>&version=<draft>   # -> codeBlockMetadataOutput.lowerCaseName
# then reference [tool:format_policy_number] in the agent's instructions (see "Binding" below).
```

**MCP-server tool** (⚠ currently 500s on create — dev3 bug):
```
POST /ai-worker-tool/mcp-server  { aiWorkerId:<id>, version:0, name:"Docs MCP",
  mcpServerUrl:"https://mcp.example.com", requestTimeout:5000, maximumTotalTimeout:30000,
  mcpServerHeaderRequest:{ Authorization:"Bearer …" } }
# response's actionMetaDataOutputList is the discovered actions (when the BE bug is fixed)
```

**Connector (SaaS app) — discover, then attach WITH a credential in the SAME create body:**
```
GET  /ai-worker-tool/integration-apps?identifiers=gmail   # note iconUrl, integrationAppTags, credentials[].authIdentifier
GET  /ai-worker-tool/app-actions?aiWorkerId=<id>&version=<draft>&appIdentifier=gmail   # -> {identifier, displayName}
GET  /ai-worker-tool/credentials?appIdentifiers=gmail     # pick a connected one -> credentialIdentifier
# attach WITH the credential + display metadata (icon, tags, per-action displayName) — one shot:
POST /ai-worker-tool/connector  { aiWorkerId:<id>, version:<draft>,
  appMetaDataInput:{ appIdentifier:"gmail", appDisplayName:"Gmail",
    appIconUrl:"https://…/gmail.png", appTags:["Communication"] },
  credentialMetaDataInput:{ credentialIdentifier:"<connected cred id>" },   # ⚠ REQUIRED — omit and it never registers
  actionMetaDataListInput:[ { actionIdentifier:"send_message", actionDisplayName:"Send Message",
    hitlMode:"AUTO_RUN" } ] }
# verify it registered: GET /ai-worker-tool?aiWorkerId=<id>&version=<draft> -> the action has a real
#   actionName + inputSchema/outputSchema and a credentialMetaDataOutput (all empty == not registered).
# NO credential yet? Do NOT attach a bare connector "to fix later" — there is no PUT path to add a
#   credential; you'd have to DELETE and re-create. Complete OAuth in the UI first, then attach.
```

**Credentials — verified live.** The backend **validates** the credential, so dummy/fake keys don't
work: `POST /ai-worker-tool/credential` returns `200 {"id":"","success":false}` (no error text) for an
invalid key.
- **`OTHER` (API-key) apps** → put the key in `data` as JSON: `data:"{\"apiKey\":\"<real key>\"}"`.
- **OAuth apps** → needs an `authorizationCode` (`code` + PKCE) from a completed consent screen;
  there's no headless way to mint that, so the OAuth handshake happens in the **Commotion UI** and the
  skill references the resulting `credentialIdentifier`.
Check `GET /ai-worker-tool/credentials?appIdentifiers=…` first and reuse an existing **connected**
credential (`"connected": true`) rather than re-authorising. **The credential must go in the connector's
create body (`credentialMetaDataInput`) — there is no PUT to add it afterwards** (the update schema has
no credential field). A connector attached without one never registers (empty
`actionName`/schemas, the red "could not be registered" banner) and can only be fixed by delete +
re-create. So don't attach a connector until a connected credential exists.

## Binding a tool to an agent (the `/` mention) — REQUIRED

Creating a tool attaches it to the **worker**, but an **agent only calls a tool its prompt
references** — exactly like knowledge. In the agent prompt editor this is the **`/` command**
("Type / to add tools and more"); over the API it is a **mention token in the agent's `instructions`**
(composed into the prompt and written the **Phase-6 way** — `PUT /aiagent/{id}` with the full body; it
renders in the UI editor). The tokens (all verified live):

| What | Token | Has id? |
|------|-------|---------|
| A tool/action | `[tool:<action name>]` | **no** — name only |
| Knowledge | `[knowledge:<name>\|id:<knowledgeId>]` | yes |
| Another agent (hand off to it) | `[agent:<name>\|id:<agentId>]` | yes |
| A variable | `[var:<name>]` | no |
| A skill | `[skill:<name>]` | **no** — name only |

⚠ **`[skill:…]` is the one token that is only a hint.** For tools, knowledge and agents the token *is*
the binding. For skills the binding is the `aiWorkerSkillIds` array on the `AiAgentRequest`; the token
stays literal in the resolved prompt and, on its own, does nothing. Write it anyway — it steers the
model to the right skill at the right moment — but always bind the id too. See
`skills-and-progressive-disclosure.md`.

**None of these tokens is string-substituted — every one becomes a tool call (verified live
2026-08-10).** The prompt the model sees contains the raw token; the runtime resolves it when the turn
needs it. `[knowledge:…]` → `search_knowledge_base`, `[skill:…]` → `read_skill`, `[tool:…]` → the
registered action itself. That is why the UI can show all of them as chips while the resolved
`finalInstructions` still shows the literal text: the chip is a display layer, not an expansion.

⚠ **This is why a knowledge token carries `|id:` and a tool token doesn't.** The observed call is
`search_knowledge_base(query, filters: [{"key": "id", "value": "<the id from the token>"}])` — the id
**is the retrieval filter**, scoping the search to that one document. A wrong or stale id doesn't
error; the search just comes back empty and the agent answers from nothing.

⚠ **Action names are re-minted per worker version.** The same custom tool was
`check-slot-availability-2135` at v1 and `check-slot-availability-2136` at v2. The platform rewrites
existing `[tool:…]` tokens (in agent prompts *and* skill bodies) when a draft is forked, so live
references keep working — but **never hand-copy an action name across versions.** Re-read
`GET /ai-worker-tool?aiWorkerId=…&version=<the version you are editing>` before writing a new one.

The `<action name>` is the tool's **action** name, not the tool name and not its id — read it from
`GET /ai-worker-tool?aiWorkerId=<id>&version=<draft>` → each tool's `actionMetaDataOutputList[].actionName`.
A custom tool named `lookup_order` gets action `lookup-order-189`; connector actions look like
`google-sheet-get-rows-686`. **Code-block tools are the exception (verified live):** their
`actionMetaDataOutputList` is empty, so read the name from `codeBlockMetadataOutput.lowerCaseName` — it is
your `name` used as-is (no numeric suffix), e.g. `[tool:format_policy_number]`. So:

```
GET /ai-worker-tool?aiWorkerId=<id>&version=<draft>        # find the action name(s)
# compose the token into instructions and PUT the agent (full body — PUT is a full replace):
PUT /aiagent/<agentId>  { …, instructions:
  "…When the caller gives an order id, look it up.\n\n[tool:lookup-order-189]\n\n…" }
```

Verified live: writing `[tool:lookup-order-189]` into an agent's `instructions` **round-trips intact and
works at runtime** (a Test-Agent run called the tool), same as the `/Knowledge` token. **`PUT` is enough
for UI visibility too** — verified live 2026-08-07 that a `PUT` renders in the prompt editor, both into a
blank editor and over an existing prompt (this replaces the older 2026-07-21 finding that a PUT left the
editor blank). Tokens show as **plain text**, not styled chips — cosmetic only. **Only the agents whose
prompt carries the `[tool:…]` mention call that tool** — that is how you scope a worker tool to a specific agent.
Built-in actions (`end_call`, `transfer_to_human`, …) are referenced by their action name the same
way; describe when to use them in the prose.

## HITL — the pause and the resume

Flag any action with `hitlMode:"REQUIRE_APPROVAL"` (vs `AUTO_RUN`, or `ASK_FOR_DETAILS` to collect
missing input). At runtime, when the worker wants to run that action it **pauses** and asks the human
to approve. Resume the run with:

```
POST /aiworker/continue  { workerId, agentId, sessionId, runId, agentRunId,
                           userInput:{ <the approval/decision> }, toolName? }
```

(The `sessionId`/`runId`/`agentRunId` come from the paused conversation, so the resume is driven by
whatever is hosting the live conversation — not by the build flow itself.) Use `REQUIRE_APPROVAL` for
anything side-effecting or risky (sending money, deleting records, contacting a customer); leave
read-only lookups on `AUTO_RUN`.

## Auto-capabilities: reasoning & state (NOT tools)

Two capabilities aren't tools — you don't attach them, you turn them on (or they appear on their own):

- **Reasoning** — a config flag, not a tool. Set it on the agent's advanced settings:
  `advancedSettingsRequest.languageModelSettingsRequest.reasoningEffortEnabled: true` +
  `reasoningEffort: "LOW" | "MEDIUM" | "HIGH"` (also available at worker LM settings). Only works on a
  model that supports it — `GET /aimodel` shows `reasoningEffortEnabled` per model. Use it for agents
  that must plan/decompose; leave it off for simple Q&A (latency/cost).
- **State access** — appears automatically once the worker *has* state. State variables are produced by
  workflow Set-State steps or captured from tool/API responses (`stateVariableTo…Map`), and an agent
  reads them in its prompt via the **`[var:<name>]`** mention (the same `/` family as `[tool:…]` /
  `[knowledge:…]`). Nothing to create — if state exists, reference it with `[var:…]`.

## A2A — calling another agent

A2A (agent-to-agent) lets one worker's agent call **another Commotion worker** as a tool. An A2A setup
has **two sides**, and they are not symmetric — one is fully API-driven, the other is not:

| Side | What it needs | Can you do it through this skill? |
|------|---------------|-----------------------------------|
| **Client** (the worker that *calls*) | an `A2A_AGENT` tool pointing at the server worker's `/a2a/{id}` endpoint, then the tool token in an agent prompt | **Yes** — `POST /ai-worker-tool/a2a-agent` |
| **Server** (the worker being *called*) | "enabled as A2A server" | **No** — no field, no endpoint. UI/internal only. |

**Do the client side, then stop and tell the user the server side is a manual step.** Attaching the
tool and saying nothing produces a setup that looks complete in the UI and fails at runtime — that is
the exact defect QA reported (2026-08-31).

### Client side — attach the A2A tool (verified live, dev3, 2026-08-31)

```
POST /ai-worker-tool/a2a-agent
{ "aiWorkerId": "<client worker id>", "version": <draft version>,
  "a2aAgentMetaData": {
    "agentName": "fx_desk",                                    # required
    "agentDescription": "FX rate desk agent. Ask it for rates.", # required — this is what the LLM reads
    "agentEndpointUrl": "<base>/a2a/<server worker id>",        # required
    "agentTags": ["fx"],
    "credentialId": "<stored A2A credential id>"                # OPTIONAL — omit when the endpoint needs no outbound auth
  } }
→ 200 { id, version, aiWorkerId, a2aAgentMetaDataOutput: { a2aAgentName: "fx-desk-1299", … } }
```

- **`credentialId` is genuinely optional** — a create with **no** credential returns `200` and a
  working tool record (verified). This is *unlike* connectors, which silently register empty without one.
- **Bind it like any other tool**: the backend returns a slugged `a2aAgentName` (`fx-desk-1299`) — put
  **`[tool:fx-desk-1299]`** in the calling agent's `instructions` and `PUT /aiagent/{id}` (full body).
  Round-tripped intact. Use the returned `a2aAgentName`, **not** the `agentName` you sent.
- `PUT /ai-worker-tool/a2a-agent/{aiWorkerToolId}` updates it. **The update replaces the whole
  metadata block** — restate `credentialId` or you drop the connection.
- Schemas: `CreateA2aAgentToolRequest` / `UpdateA2aAgentToolRequest` / `A2AAgentMetaDataInput`.
  `A2AAgentMetaDataInput` also carries `mcpAuthType` (`BASIC_AUTH|BEARER_AUTH|CUSTOM_HEADERS_AUTH`)
  with `basicAuth` / `bearerToken` / `customHeadersAuths` for an endpoint that authenticates inline.

### Server side — the gap you must surface (verified live 2026-08-31)

`POST /a2a/{workerId}` on a worker that has not been enabled as an A2A server returns **HTTP 200**
with the JSON-RPC error `"Worker is not enabled as A2A server"`.

**Deploying is not the toggle.** A freshly created worker was given an enabled agent and
`POST /aiworker/{id}/deploy` → `LIVE`, and `POST /a2a/{id}` *still* returned
`"Worker is not enabled as A2A server"`. The card at `GET /.well-known/agent.json/{id}` returns `200`
either way, so **the card is not proof the server works** — it is served for any worker, enabled or not.

There is no `a2aServerEnabled` (or equivalent) anywhere in the spec: not on `AiWorkerRequest`, not on
`AiAgentRequest`, not on `AiAgentTriggerInput` (whose `aiAgentTriggerType` enum is only
`CHANNELS|WEBHOOKS|EVENTS` and `aiAgentChannelType` only `VOICE|CHAT`), and there is no
`/aiworker/**` path for it. A/B against a worker that *is* enabled showed the two records differ in
**nothing** the API exposes. The flag lives outside the public API.

### The verification gate — never declare an A2A setup done without it

After attaching the client tool, **probe the server**:

```
POST /a2a/<server worker id>
     { "jsonrpc":"2.0", "id":"1", "method":"message/send",
       "params": { "message": { "role":"user",
                                "parts":[{"kind":"text","text":"ping"}], "messageId":"m1" } } }
```

- `result.artifacts[…].parts[…].text` → the server side is live; the setup works end to end.
- `error.message == "Worker is not enabled as A2A server"` → **say so explicitly**, name the server
  worker, and tell the user this one switch must be flipped in the Commotion UI (or by a platform
  admin) because the API does not expose it. Do not report the setup as complete.

## Where this sits in the create-worker flow

Attach tools **after the agent(s) are provisioned and before the deploy gate** (SKILL.md Phase 8).
Create each tool on the worker draft, then **reference it in the prompt of the agent(s) that should
use it** (the `/Actions` mention, like `/Knowledge`) — that's how you scope which member calls what on
a `MULTI_AGENT` worker. Show the user each tool (and each HITL gate) before you write it, then verify
with `GET /ai-worker-tool?aiWorkerId=<id>&version=<draft>` before deploying.

## Verified live (worker `6a379970421f279076ad4668`, draft v2)

A real run that attached a custom tool + a built-in action and created an "Order Concierge" agent:

- **Custom tool** `lookup_order` → 200; backend returned action id `lookup-order-189`, `hitlMode: null`.
- **Built-in** `transfer_to_human` → 200; first attempt also sent `end_call` → `400 "already configured"`
  (it's a default). Built-in entries carried **no** `hitlMode`.
- **Code-block** (separate probe) → `/code-block/run` with `result=2+2;print(result)` → `200
  {success:true, stdout:"4", …}`. `POST /ai-worker-tool/code-block` (name `format_policy_number`,
  `language:"PYTHON"`) → 200 with `codeBlockMetadataResponse`. `GET /ai-worker-tool` → `toolType:"CODE_BLOCK"`,
  `actionMetaDataOutputList:[]` (empty), name in `codeBlockMetadataOutput.lowerCaseName` → bind
  `[tool:format_policy_number]`. `hitlMode` accepted (null when omitted); `toolUsageTypes` defaulted `["LLM"]`.
- **MCP-server** → `500` on every attempt: public DeepWiki `/mcp` + `/sse`, **and** a reachable
  known-good `/mcp`. Decisive: a direct `POST /mcp initialize` to that same URL returns **200 with a
  valid MCP result** (proven from outside), yet create still 500s — so reachability, auth, protocol,
  trailing-slash, and input are all ruled out. **Definitively a dev3 backend bug in `mcp-server` create.**
- **A2A** *(superseded — re-probed 2026-08-31, see the A2A section above)* — the **client** half is
  now fully API-driven: `POST /ai-worker-tool/a2a-agent` → `200` with a slugged `a2aAgentName`, no
  credential required, and `[tool:<a2aAgentName>]` round-trips into the calling agent's prompt. The
  earlier "no attach-as-tool endpoint" note is **wrong** and has been corrected. The **server** half
  is still closed: `POST /a2a/{id}` → `"Worker is not enabled as A2A server"` even after the worker is
  deployed `LIVE`, and no enable field/path exists in the spec.
- **Connector** — `integration-apps` → 50 apps; `app-actions` (clockify) → `create_client` …;
  `POST /ai-worker-tool/connector` with that action and **no credential** → 200
  (`hitlMode:REQUIRE_APPROVAL` round-tripped). `POST /ai-worker-tool/credential` with a dummy Clockify
  key → `200 {"id":"","success":false}` (keys are validated — dummies don't take).
- **Connector must carry a credential at create (worker `6a63138755ccadc213ea91c3`, HR Voice Assistant,
  2026-07-24).** A worker built via the skill attached Gmail + Google Calendar with **no
  `credentialMetaDataInput`** → tools listed `200` but with empty `actionName`/`inputSchema`/`outputSchema`
  and no `credentialMetaDataOutput`; the UI showed *"Tools could not be registered — MCP server failed to
  register, or the MCP authentication token is missing."* A/B on a throwaway worker: same connector
  **without** a credential → empty action fields (not registered); **with** a connected
  `credentialIdentifier` → real `actionName` (`gmail-send-message-426`) + discovered schemas +
  `credentialMetaDataOutput` (registered). `UpdateConnectorToolRequest` has **no credential field** →
  can't be fixed by PUT; must delete + re-create. Display metadata is separate and also POST-only:
  passing `appIconUrl` + `appTags` (from `integration-apps`) and per-action `actionDisplayName` (from
  `app-actions`) makes the tool render with the real icon / **Category** / action chips instead of a
  generic "G" and blank columns. `actionDescription` was accepted but **not persisted** (returned `""`).
  (Note `mcpServerIdentifier`/`mcpServerUrl` stay empty for connectors either way — those belong to the
  external `mcp-server` tool type, not connectors; don't use them as the registration signal.)
- **Binding** — embedding `[tool:lookup-order-189]` in the Order Concierge agent's `instructions`
  round-tripped intact at the API, wiring the custom tool to that agent (`PUT /aiagent/{id}` is enough —
  the prompt renders in the UI editor). Token vocabulary across
  the workspace (a 500-agent scan): `[tool:…]` ×540 (name only, no id), `[knowledge:…|id:…]` ×280,
  `[var:…]` ×390, `[agent:…|id:…]` ×90.
- Error bodies are **XML** (`<LinkedHashMap>…`), not JSON. Tools attach only on the **draft** (v2);
  the live worker was v1.
