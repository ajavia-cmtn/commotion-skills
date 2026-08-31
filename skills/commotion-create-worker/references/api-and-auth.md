# API & transport — the two Commotion MCP tools

This is the single "how to call it" reference. Every platform action is one call through the
connected **Commotion MCP** server, which holds the credential and reaches the Commotion backend for you.
The domain reference files (`aiworker-lifecycle.md`, `agents-and-orchestration.md`, …) describe the
*behavior*; this file is the *transport*.

## The two tools

- **`commotion_request`** — one authenticated call. Arguments:
  `{ "method": "GET|POST|PUT|DELETE", "path": "/…", "body": <JSON, for writes> }`. `path` starts
  with `/` and may carry a query string (e.g. `/aiagent?workerId=ID&version=0`); the base URL is
  fixed server-side, so pass a **path, not a URL**. Returns `{ "status": <http status>,
  "body": <parsed JSON | text | null> }` — a non-2xx is **returned, not thrown**, so read the status
  and body and adjust. `body` is a tool argument, so there are no temp files for request payloads.
- **`commotion_schema`** — a bundled request schema. Arguments:
  `{ "schema_name": "AiWorkerRequest", "refresh": false }`. Returns the named OpenAPI component
  bundled self-contained with its `$defs` (refs rewritten to `#/$defs/…`). Any component name in the
  live spec works, not just those listed below; the spec is cached server-side after the first call.
  **Never invent a field that isn't in the schema.**

Auth is handled entirely by the MCP server — you never pass a token (see **Auth** below).

Read ids straight from a result — after `POST /aiworker` the new id is `body.id`; from a list call
it's `body[0].id`. Feed that id into the next call's `path`. (No shell, no `jq` — you read the JSON
the tool returns.)

## Auth — automatic browser login (OAuth)

Auth is owned by the MCP client + server; you never handle a credential or token. On first use of the
Commotion MCP, the client opens a Commotion login in the browser (OAuth 2.1 authorization-code +
PKCE); the user signs in once, and the client then attaches the resulting token as
`Authorization: Bearer` on **every** call automatically. The MCP server validates/forwards it to the
Commotion backend — the raw token never enters the conversation.

So: **never** ask the user for an email/password, **never** pass a `token` argument, and never reuse
another user's token. The backend attributes actions to the signed-in user (e.g. a created worker's
`createdByUserId` is their email). If the two tools aren't available at all, the Commotion MCP isn't
connected — ask the user to add/authorize it via `/mcp` (select that environment's connector →
Authenticate).

**One connector per environment.** Each Commotion environment has its own connector exposing its own
copy of these tools, so `commotion_request` exists once per environment under a different namespace.
The exact tool name depends on the client — `mcp__commotion-tcuat__commotion_request` in Claude Code,
`mcp__claude_ai_Commotion_Agent_UAT__commotion_request` in the Claude desktop app, where the name
comes from whatever the administrator called the connector. Identify the environment by the
environment word in the name, not by a fixed prefix; pick one (see "Pick the environment before the
first tool call" in the SKILL) and use it for every call in the task. A `401` from one connector means
it is not authorized — never a reason to retry on another environment. Where this file writes
`commotion_request` unprefixed, it means the tool on the environment you selected. (Swagger UI for humans, per environment:
`https://api-tier0.<env>.gocommotion.com/swagger-ui/index.html`.)

## Error semantics

`commotion_request` returns `{ "status", "body" }` for every call — a non-2xx is **not** thrown, it
comes back with the backend body (sometimes XML, e.g. `<LinkedHashMap>…`, not JSON). Read the status
+ message and check it against the relevant reference's "edges/golden rules" before retrying — most
failures are a known gotcha (missing `version` on a PUT, an action re-added, a live-only retrieve on
a draft, …). Only a transport failure (backend unreachable) comes back as a tool error.

## Never discover a shape by writing (this is what gets you blocked)

**`commotion_schema` is the only way to find out what a body looks like. Never send a write to find out.**

A "probe" write — a `POST`/`PUT` carrying placeholder text (`"SHAPE PROBE - do not deploy"`, `"test"`,
`"x"`) to see which fields the backend accepts — is not a read. On this API it is a **destructive
overwrite**, because `PUT /aiagent/{id}` and `PUT /aiworker/{id}` are **full replaces**: the probe
body *becomes* the record. A probe against a real worker silently blanks its prompt, its model config
and its guardrails. QA hit exactly this on 2026-08-31 (`PUT /aiagent/6a7b…` on "TCPL Field Saathi Two
Way" with `"instructions": "SHAPE PROBE - do not deploy"`), and the client's permission layer stopped
it — correctly.

So, in order:

1. **Shape** → `commotion_schema { "schema_name": "AiAgentRequest" }`. Any component in the live spec
   works. Never invent a field that isn't in it.
2. **Values / defaults** → `GET /aiworker/metadata`, `GET /aimodel?pageSize=200`,
   `GET /ai-worker-tool/metadata`. All reads.
3. **What a real record looks like** → `GET` an existing one and read its fields. A read is free and
   tells you more than a failed write.
4. **Still unsure after all three?** Say so and ask the user. Do not resolve it with a write.

If you genuinely need to try a write shape, **create a throwaway worker** (`POST /aiworker`, name it
`ZZ-TEST-…`), probe on *that*, and `DELETE /aiworker/{id}` when done. Never on a record you did not
create in this task.

**Corollary — never send filler into a real field.** Every value in a write body must be the value
you actually intend to persist. `"TBD"`, `"do not deploy"`, `"placeholder"` and empty strings are all
data as far as the backend is concerned.

## When a call is refused by the client, not by the backend

There are two different kinds of "no", and they need opposite responses:

| Signal | Who said no | What it means |
|--------|-------------|---------------|
| `commotion_request` returns `{ "status": 4xx/5xx, … }` | the **backend** | a real API error — read the body, check the reference's gotchas, adjust |
| the tool call never runs — *"Permission for this action was denied"*, *"Blocked by classifier"*, a permission prompt | the **client** (Claude Code / desktop app) | the call was judged too risky to run unattended |

A client-side denial is **not** an API error and never a reason to retry — not with the same body, not
on another connector, and above all **not by reaching for `Bash`/`curl` to make the same call another
way**. That is working around the user's own safety setting.

**This includes handing the user a script.** Generating an `apply.py` / `curl` snippet and telling them
to `export COMMOTION_<ENV>_TOKEN=… && python apply.py --write` is the same bypass with an extra step,
and it is worse in three specific ways: the bearer token leaves the MCP server and lands in the user's
shell history and environment (the whole point of OAuth-in-the-MCP is that it never does); the call
stops appearing in the MCP's audit log, which records `user / method / path / status` for every
`commotion_request`; and **you do not know the base URL** — it is fixed server-side and deliberately not
exposed to you, so any URL in that script is a guess. A script built on a guessed base URL is not a
fallback, it is a defect you handed to someone else to run against a shared environment.

Staging the *content* is fine and often right — write the patched prompt to a file, show the diff, keep
the original for revert. Just stop at the boundary: the **write** goes through `commotion_request` once
the user approves it, or it does not happen.

Do this instead:

1. **Re-read your own body first.** In practice the denial is usually right: a full-replace `PUT`
   carrying placeholder text, a write to a worker you did not create, or a write against a
   **production** connector. Fix the call.
2. If the call is legitimate, **stop and tell the user in one plain sentence**: what you were about
   to do, which tool was blocked, and that they can approve it or allow it in settings. Then continue
   with everything that does not depend on it.
3. Recurring denials on legitimate writes are a **setup** problem, not something to out-argue — point
   the user at the permissions section in the repo README. One `permissions.allow` rule fixes it for
   good; a bespoke script fixes it once, unsafely.

## Never conclude "the API doesn't expose that" from a path you guessed

A wrong `404` is more dangerous than a wrong write, because it looks like a finding. The failure mode,
seen in the wild on 2026-08-31: a session wanted to know which knowledge base a UAT worker's tool
pointed at, tried **`/customtool`** and **`/aiagent/{id}/tools`** — *neither of which exists* — and
concluded "tool wiring isn't exposed", then verified the claim against a **different environment's**
data instead. Both reads it wanted are fully available:

| Question | Real path |
|---|---|
| what tools does this worker have, and how are they configured? | `GET /ai-worker-tool?aiWorkerId=<id>&version=<n>` |
| what knowledge is attached to this worker? | `GET /aiworker/knowledge?aiWorkerId=<id>` |

Note the shape of the mistake: both guesses were **agent**-scoped. Tools and knowledge hang off the
**worker** (`aiWorkerId` + `version`) — there is no agent↔tool or agent↔knowledge field on the API at
all. An agent is wired to a tool by the `[tool:…]` / `[knowledge:…|id:…]` token in its `instructions`,
so "which KB does this agent use?" is answered by reading the **prompt** plus the worker's knowledge
list, not by a tools sub-resource that does not exist.

So, before writing "the API doesn't support X":

1. Check the endpoint map in this file and the domain reference for that surface.
2. Check the live spec — `commotion_schema` resolves any component name in it.
3. Only then say it, and say **what you checked**: "no path under `/aiworker/**` and no field on
   `AiWorkerRequest`" is a finding; "I tried two paths I made up and got 404" is not.

## Never use one environment's data as evidence about another

Related, and equally load-bearing: if the task is about **tcuat**, a number measured on **dev3** is not
a weaker version of the answer — it is a different answer to a different question. They are separate
backends holding unrelated data.

When you cannot read what you need in the selected environment, the honest move is to say the check is
**unavailable there** and let the user decide. Do not substitute another environment and caveat it; a
caveat under a confident-looking number gets skimmed, and the number gets quoted. If you have already
done it, lead with the substitution rather than trailing it: *"I could not read this on tcuat; the
figure below is dev3 and may not transfer."*

## Untrusted-id safety

Any id you interpolate into a path must be a safe segment (`^[A-Za-z0-9_-]+$`). Ids returned by the
backend already satisfy this; don't pass user free-text into a path.

## List-response shape

List endpoints return a bare JSON array today. Tolerate a paged wrapper too — if a response is an
object, the records may be under `content` / `items` / `data` / `results`. Parse defensively with `jq`.

## Endpoint map

Paths are relative to the base URL. "Schema" is the `commotion_schema` name for the request body.

### Workers & models
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/aiworker` | list workers (live + draft) | — |
| GET | `/aiworker/{id}` | retrieve **live** worker (`?version=N` for a version) | — |
| POST | `/aiworker` | create worker → DRAFT v0 | `AiWorkerRequest` |
| PUT | `/aiworker/{id}` | update draft (full PUT; body needs `version`) | `AiWorkerRequest` |
| POST | `/aiworker/{id}/deploy?version=N` | deploy version N → LIVE | — |
| POST | `/aiworker/{id}/draft?version=N` | save/keep as draft (revert live → new draft) | — |
| GET | `/aiworker/{id}/versions` | version history (status LIVE/DRAFT) | — |
| POST | `/aiworker/continue` | resume a HITL-paused run | `CopilotChatContinueInput` |
| GET | `/aiworker/metadata` | valid config values/defaults | — |
| GET | `/aimodel` | supported models (modelCode/providerCode/id) | — |
| POST | `/aiworker/run` | run the worker in text — TEST it (returns `{response,status,...}`) | `AiWorkerRunRequest` |

`AiWorkerRunRequest`'s schema-required fields are `workerId` + `messageText`, but two more conditions
apply at runtime (verified live): (1) you must pass an **identity** — one of `userId` /
`fingerprintId` / `audienceId` — or the call `400`s with *"At least one of userId, fingerprintId, or
audienceId must be provided"*; and (2) the **worker must be deployed (LIVE) with an enabled agent** —
running a draft returns `status:"FAILED"` *"Worker is not available…"*, and deploying a `SINGLE_AGENT`
worker with no enabled agent `400`s. Reuse `conversationId`/`sessionId` across turns. Parse the response
tolerantly (the body can contain raw newlines) and retry on 5xx (the endpoint is occasionally flaky).
Use this to evaluate prompt adherence and hallucination before handoff. **Caveat (known code-side bug):
when a guardrail intercepts a turn, the run currently returns `status:"FAILED"` with a generic *"An error
has occurred … reference number …"* message instead of the configured fallback text — a backend issue,
not intended behaviour (see `control-and-reliability.md`). Verify guardrail UX in the delivered channel,
not here.**

### Agents
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/aiagent?workerId=&version=&pageNumber=0&pageSize=10&sortDirection=DESC` | list agents | — |
| GET | `/aiagent/{id}?version=N` | retrieve one agent | — |
| POST | `/aiagent` | create agent on a draft worker | `AiAgentRequest` |
| POST | `/aiagent/standard` | create a *standard* agent (e.g. FAQ) | `CreateStandardAgentRequest` |
| PUT | `/aiagent/{id}` | update an agent in place (prompt, name, enable/disable, model) — **renders in the UI editor**; keeps the agent id. Full replace: resend kept fields | `AiAgentRequest` |
| DELETE | `/aiagent/{id}?version=N` | delete an agent (`version` query param required) | — |

### Skills (`ai-worker-skill`) — instruction blocks the agent loads on demand
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/ai-worker-skill?aiWorkerId=&version=` | list a version's skills (id field is `aiWorkerSkillId`) | — |
| POST | `/ai-worker-skill` | create a skill on a draft | `AiWorkerSkillRequest` |
| PUT | `/ai-worker-skill/{aiWorkerSkillId}` | full update (keeps the id) | `AiWorkerSkillRequest` |

`(worker, version)`-scoped. Bound to an agent through `aiWorkerSkillIds` on the `AiAgentRequest`, which
puts a **name + description manifest** in the system prompt and lets the agent pull each body in via a
built-in `read_skill(name)` tool — full behaviour in `references/skills-and-progressive-disclosure.md`.

### Knowledge & files
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/aiworker/knowledge?aiWorkerId=&pageNumber=&pageSize=&knowledgeType=&knowledgeStatus=&sortDirection=` | list a worker's knowledge (poll `aiWorkerKnowledgeStatus`) | — |
| GET | `/aiworker/knowledge/{id}` | retrieve one item | — |
| POST | `/aiworker/knowledge/bulk` | create item(s) (array body) | `CreateAiWorkerKnowledgeItemRequest` (per item) |
| POST | `/aiworker/knowledge/by-global/{globalId}?aiWorkerId=` | attach a global KB | — |
| GET | `/aiworker/knowledge/global?pageNumber=&pageSize=&sortDirection=` | global-KB catalogue | — |
| POST | `/aiworker/knowledge/index` | index items (array of ids; sync→bool) | — |
| PUT | `/aiworker/knowledge/{id}` | rename an item | `UpdateAiWorkerKnowledgeNameRequest` |
| DELETE | `/aiworker/knowledge` | delete items (array of ids in body) | — |
| POST | `/aiworker/file-upload/text` | upload inline text | `CreateAndUploadTextFileRequest` |
| POST | `/aiworker/file-upload/url` | presigned upload URL for a document | `FileUploadUrlRequest` |
| DELETE | `/aiworker/file-upload/delete` | delete uploaded files | `FileDeleteRequest` |

The byte PUT to the returned `preSignedUrl` is **not** through Kong — `curl -X PUT --upload-file
<file> -H 'x-ms-blob-type: BlockBlob' "<preSignedUrl>"` (Azure Blob Storage; success is `201`).

### Tools (`ai-worker-tool`) & connectors
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/ai-worker-tool?aiWorkerId=&version=&aiWorkerToolId=&searchText=&pageNumber=&pageSize=&sortDirection=` | list a worker's tools / one tool | — |
| DELETE | `/ai-worker-tool` | delete tools (array of ids in body) | — |
| GET | `/ai-worker-tool/metadata` | built-in action catalog | — |
| POST / PUT | `/ai-worker-tool/custom-tool[/{id}]` | custom HTTP-wrapper tool | `CreateCustomToolRequest` |
| POST / PUT | `/ai-worker-tool/built-in-actions[/{id}]` | built-in actions tool | `CreateBuiltInActionsToolRequest` |
| POST / PUT | `/ai-worker-tool/code-block[/{id}]` | sandboxed Python tool | `CreateCodeBlockToolRequest` |
| POST | `/ai-worker-tool/code-block/run` | test-run source in the sandbox (stateless) | `RunCodeBlockRequest` |
| POST / PUT | `/ai-worker-tool/mcp-server[/{id}]` | external MCP-server tool (⚠ create 500s — dev3 bug) | `CreateMcpServerRequest` / `UpdateMcpServerRequest` |
| POST / PUT | `/ai-worker-tool/connector[/{id}]` | SaaS connector tool | `CreateConnectorToolRequest` / `UpdateConnectorToolRequest` |
| POST | `/ai-worker-tool/credential` | store a connector credential | `CreateCredentialRequest` |
| GET | `/ai-worker-tool/credentials?appIdentifiers=clockify&appIdentifiers=slack` | list stored credentials | — |
| DELETE | `/ai-worker-tool/credential` | delete credentials (body `{"credentialIds":[…]}`) | — |
| GET | `/ai-worker-tool/integration-apps?identifiers=&pageNumber=&pageSize=` | available SaaS apps | — |
| GET | `/ai-worker-tool/app-actions?aiWorkerId=&version=&appIdentifier=&searchText=&pageNumber=&pageSize=` | an app's actions | — |
| GET | `/ai-worker-tool/webhooks?appIdentifier=&searchText=&pageNumber=&pageSize=` | an app's webhooks | — |

### Settings — pronunciation dictionaries & state variables (worker-scoped resources)
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/ai-pronunciation-dict?workerId=&version=&pageNumber=&pageSize=&sortDirection=` | list pronunciation entries | — |
| GET | `/ai-pronunciation-dict/{pronunciationDictId}?version=` | one entry (id field is `pronunciationDictId`, not `id`) | — |
| POST / PUT | `/ai-pronunciation-dict[/{pronunciationDictId}]` | create / update an entry | `AiPronunciationDictRequest` |
| DELETE | `/ai-pronunciation-dict` | bulk delete (array body) | `DeleteAiPronunciationDictRequest` |
| GET | `/ai-worker-variable-schema?workerId=&version=&pageNumber=&pageSize=&sortDirection=` | list state variables | — |
| GET | `/ai-worker-variable-schema/{variableId}` | one variable (id field is `id`) | — |
| POST / PUT | `/ai-worker-variable-schema[/{variableId}]` | create / update a variable | `AiWorkerVariableSchemaRequest` |
| DELETE | `/ai-worker-variable-schema` | bulk delete (array body) | `DeleteAiWorkerVariableSchemaRequest` |

Both are `(worker, version)`-scoped, created on a draft; full behaviour in
`references/settings-variables-pronunciation.md`.

### Knowledge settings (indexing / embedding / chunking — the `km-setting` plane)
| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/aiworker/km-setting/metadata` | valid indexing/embedding/chunking options | — |
| GET | `/aiworker/km-setting/{entityId}` | a worker's KM setting (returns `settingId`, `entityId`, config) | — |
| PUT | `/aiworker/km-setting/setting/{settingId}` | update the KM setting (get `settingId` from the retrieve) | `KMSettingUpdateRequest` |

Auto-provisioned per worker; see `references/knowledge-and-rag.md`.

### A2A (agent-to-agent — a separate protocol)
| Method | Path | Purpose |
|--------|------|---------|
| GET | `/.well-known/agent.json/{workerId}` | the worker's advertised agent card |
| POST | `/a2a/{workerId}` | send a JSON-RPC message to the worker |

## Schema names for `commotion_schema`

`AiWorkerRequest`, `AiAgentRequest`, `AiWorkerSkillRequest`, `CreateStandardAgentRequest`, `CreateAiWorkerKnowledgeItemRequest`,
`UpdateAiWorkerKnowledgeNameRequest`, `CreateAndUploadTextFileRequest`, `FileUploadUrlRequest`,
`FileDeleteRequest`, `CreateCustomToolRequest`, `CreateBuiltInActionsToolRequest`,
`CreateCodeBlockToolRequest`, `RunCodeBlockRequest`,
`CreateMcpServerRequest`, `UpdateMcpServerRequest`, `CreateConnectorToolRequest`,
`UpdateConnectorToolRequest`, `CreateCredentialRequest`, `CopilotChatContinueInput`,
`AiPronunciationDictRequest`, `AiWorkerVariableSchemaRequest`, `KMSettingUpdateRequest`.
(Any other component name in `/v3/api-docs/public` works too.)
