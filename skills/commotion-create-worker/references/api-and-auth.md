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
