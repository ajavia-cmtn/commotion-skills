# Call Analyzer — the observability plane (endpoint map + gotchas)

The transport contract and endpoint map for `commotion_analyzer`, the read-only tool that fetches what
**actually happened** on an interaction. This is the **canonical map for the Call Analyzer plane** — it
lives here because `commotion-debug` introduced the plane, but it is **not debug-only**:
`commotion-run-evals`, `commotion-improve-worker`, `commotion-generate-scenarios` and
`commotion-create-worker` cross-link here too. It is the third companion to the worker-domain map in
`commotion-create-worker/references/api-and-auth.md` and the eval-domain map in
`commotion-generate-scenarios/references/eval-domain-api.md`.

**Why the loop skills want it, not only the debug skill.** The eval surface tells you *that* a scenario
failed plus the evaluator's opinion of why. Call Analyzer tells you what the worker actually did: the
turn-by-turn transcript, whether each tool fired and what it returned, real per-turn latency, whether the
simulated caller ever spoke, whether the context sent to the model was intact. **A simulation is just a
real call with a robot caller** — every signal here applies to a scenario-run exactly as it does to
production traffic, and several rows of `commotion-improve-worker`'s failure→fix taxonomy are only
*provable* from this plane.

⚠ **Outside `commotion-debug`, treat it as optional enrichment.** The MCP server registers
`commotion_analyzer` only in authenticated mode, so it may be absent; and even when present, the
connected user may lack access (`{ "status": 403 }`) or see a role-reduced view. In the loop skills,
treat **all three** — tool absent, `403`, or masked/partial data — the same way: say so once and fall
back to `evaluationReasoning`, never block a run on it. In `commotion-debug` it is essential: tool
absence, or a `403` on the core call read, is a stop condition.

This is a **second plane**, separate from the dev3 BE REST plane. Call Analyzer holds what happened; it
does **not** hold the worker — every change goes through `commotion_request` on the BE plane.

There is **no OpenAPI spec** for this plane, so `commotion_schema` does not cover it and this file *is*
the map. Everything below was probed live against dev3 on 2026-07-27/28; sizes are real measurements.

## The tool

**`commotion_analyzer`** — one GET: `{ "path": "/api/…", "max_bytes": <optional> }` →
`{ "status": <http status>, "body": <parsed JSON|text|null> }`, plus `"truncated": true` and
`"dropped": { "<key>": <n> }` when the response had to be shortened. A non-2xx is **returned, not
thrown** — read the status and adjust.

- **Read-only by construction.** There is no `method` argument; nothing here can change production.
  (The gateway also rejects non-GET verbs with `404`.)
- **Token-based, per-user RBAC.** The MCP forwards your connected user's token; Call Analyzer resolves
  your role from it. You never pass or ask for a token. Two consequences to handle:
  - **`403` = no Call Analyzer access.** If a call returns `{ "status": 403 }`, the connected user has
    no `callAnalyzer` grant. Stop and say so (see the permission-denied branch in `SKILL.md`); do not
    retry or treat it as a missing call.
  - **Data is role-scoped.** `admin` sees everything cross-workspace; `analyst`/`reader` get a
    reduced/redacted view (some fields masked, some endpoints `403`). A smaller-than-expected payload
    may be your role, not missing data — this is distinct from `truncated`, and re-querying narrower
    will not recover fields your role can't see.
- **Pass a path, not a URL** — the base URL is fixed server-side. Query strings are allowed.
- **`truncated` means incomplete evidence.** Re-query narrower (fewer `fields=` sections, a shorter log
  window, a line filter) rather than reasoning from the fragment that survived. `dropped` names exactly
  which list was shortened and by how many.

## ⚠ Never surface Commotion's upstream vendors — this plane leaks them (verified live 2026-08-06)

**This is the canonical rule for every Commotion skill.** Commotion's first-party models are served
under Commotion names (`commotion-3.6-35b`, `commotion-omni`, `commotion-laya-v1-5`, "Commotion
Guard"). Internally they run on third-party bases, and **the Call Analyzer plane reports those
internal identifiers, not the Commotion names.** Repeating them to a customer discloses Commotion's
supply chain. Verified on one live call:

| Where it appears | What the plane returns | What you say instead |
|---|---|---|
| `usageStats.llm_usage_by_model[].model` | `qwen3.6-35b` | `commotion-3.6-35b` |
| `usageStats.llm_usage_by_model[].model` | `gemma-4-31b-it` | `commotion-4-31b-it` |
| `usageStats.llm_usage_by_model[].provider` | `CommotionLLMService#524` | Commotion |
| `stt_segment_audio[].sttModel` | `qwen3-omni` | `commotion-omni` |
| `usageStats.audio_processing_by_model[].model` | `krisp` (vad / noise_filter) | Commotion noise cancellation / VAD |
| litellm errors in `errors[]` and `/logs` | `openai/qwen3-next`, `sk-…` | "the workspace's default Commotion model" |
| guardrail config | `toxicityDetectionModel: "QWEN3_GUARD"` | "Commotion Guard" (the UI's own label) |
| `tts_usage_by_model[].model` | `commotion-laya-v1-5` | *(already clean — TTS is the exception)* |

**The pattern, so you can map a name this table doesn't list.** The analyzer substitutes the upstream
vendor for the `commotion` prefix and otherwise keeps the model string: `commotion-3.6-35b` →
`qwen3.6-35b`, `commotion-4-31b-it` → `gemma-4-31b-it`, `commotion-omni` → `qwen3-omni`. So **any
model name on this plane that does *not* start with `commotion-` while the worker's config says it
should is an upstream identifier — map it back to whatever the worker's own
`workerLLMConfigurationResponse` / `workerTranscriptConfiguration` calls it.** That config read is the
authoritative source for the customer-facing name; never invent one.

**The rule.** Map every model/provider identifier back to its Commotion-facing name before it reaches
the user — in prose, tables, quoted errors and quoted log lines alike. The `commotion-*` codes on the
**BE plane** (`/aiworker`, `/aimodel`, `/aiworker/metadata`) are already the customer-facing names and
are always safe to quote; it is the **analyzer plane and the pipeline logs** that carry the upstream
names. When you must quote an error verbatim for accuracy, redact the model token: *"an
`AuthenticationError` on the worker's LLM credential"* carries the whole diagnosis without the vendor.

**⚠ BYOK is the opposite case — name those providers freely.** When the *customer* brings their own
key, the provider is theirs and naming it is required to be useful. Tell the two apart by ownership,
not by whether the string looks like a vendor:

| | First-party Commotion | BYOK (customer's own) |
|---|---|---|
| How to spot it | `/aiworker/metadata` → `isThirdParty: false`; provider label "Commotion"; `AiModelResponse.credentialId` is null | `isThirdParty: true`; a real `credentialId` / `voiceProviderCredentialId` |
| Examples | `commotion` (LLM), `commotion-tts`, `commotion-llm` / `commotion-asr` (STT) | `azure_openai`, `openai`, `anthropic`, `eleven-labs`, `cartesia`, `sarvam`, `smallest-ai` |
| Naming | **Commotion names only** | **Name them plainly** — "your Eleven Labs credential", "your Azure OpenAI `gpt-4o-india`" |

A simulated caller's voice is BYOK by design (personalities carry `voiceProvider: eleven-labs` with a
real `voiceProviderCredentialId`), so "the tester bot speaks through your Eleven Labs voice" is correct
and useful. "The worker's brain is qwen3.6-35b" is not.

## Payload sizes — read this before your first call

Measured on a **28-second** call. A five-minute call is proportionally larger.

| Endpoint | Size | Consequence |
|---|---|---|
| `/api/call/<id>` (no `fields=`) | ~1 KB | always safe — the Phase 0 read |
| `?fields=transcript,metrics,analysis` | ~12 KB | **the default Phase 1 fetch** |
| `?fields=config` | ~41 KB | fetch **alone**, and only when the prompt/tool wiring is implicated |
| `?fields=timeline` | ~17 KB | fetch alone |
| `?fields=` all five | ~60 KB | ⚠ **gets truncated** — don't |
| `/vad-plot` | ~79–115 KB raw | safe: the point series is dropped server-side, ~600 B arrives |
| `/logs` | **1.6–3.9 MB** | ⚠ always narrow the window *and* add a filter |
| `/api/calls-grouped` | ~388 KB | ⚠ avoid — use `/api/calls` with filters |
| `/api/chat/config` | ~283 KB | ⚠ avoid — `/api/chat/session/<id>` already carries the resolved config |

## Voice endpoints

| Path | Params | What it gives you |
|---|---|---|
| `/api/calls` | `callId`, `workerId`, `agentId`, `requestId`, `phoneNumber`, `orgId`, `workspaceId`, `date`, `timeWindow`, `sortOrder`, `page`, `limit` (≤100) | the call list — find candidates, count a pattern |
| `/api/call/<id>` | `fields=` (below) | the call itself; the RCA payload |
| `/api/call/<id>/logs` | `start`, `end` (**both required**), `session_id`, `q`, `filter`, `label`, `level`, `direction` | the pipeline trace from Loki |
| `/api/call/<id>/logs/labels` · `…/labels/<name>/values` | — | what you can filter on, before you query |
| `/api/call/<id>/raw-inbound/gap-analysis` | — | `audioLossPercent`, `avgJitterMs`, `longestGapMs`, `delays[]` with positions and severity |
| `/api/call/<id>/stt-segment-audio/summary` | — | per-turn ASR segment alignment, `packetMappingStatus` |
| `/api/call/<id>/raw-inbound/summary` | — | the inbound capture manifest (was audio even captured) |
| `/api/call/<id>/supervisor/summary` | — | which supervisor artifacts exist for this call |
| `/api/call/<id>/vad-plot` | — | VAD thresholds, `point_count`, duration-mismatch warnings |
| `/api/reports` · `/api/reports/<id>` · `/api/reports/latest` | — | the **fleet baseline**: `stop_reasons`, error/success rate, latency percentiles, per-tool metrics |

### `fields=` → what arrives

Sections are `transcript`, `timeline`, `metrics`, `analysis`, `config`. With no `fields=` you get the
shell only.

| Section | Keys |
|---|---|
| _(shell)_ | `callId`, `requestId`, `workerId`, `agentId`, `agentMapping`, `orgId`, `workspaceId`, `status`, `stopReason`, `disconnectReason`, `disconnectAttempted`, `wasConnected`, `duration`, `callMode`, `phoneNumber`, `recordingFilename`, `feedbackSummary` |
| `transcript` | `transcription[]` — turns with `role` (`user` / `assistant` / `state_snapshot`), `content`, `maskedContent`, `timestamp`, `playbackSeconds`; plus `llmFallbackEvents[]`, `playbackContract`, `hasMaskedTranscript` |
| `timeline` | `turnTimeline[]` (`turnNumber`, `relativeSeconds`, `alignment`, `hasAudio`), `annotationTimeline[]` (`matchType`, `probableTranscript`), `dataWarnings[]`, `turnAudioClips` |
| `metrics` | `latencyMetrics{summary, latency.ttfb, latency.processing, raw.pipeline, usage}`, `audioMetrics` (`silenceRatioPercent`, `botRatioPercent`, turn counts and averages), `connectionTiming`, `serviceMetrics`, **`toolCallMetrics[]`** (`function_name`, `arguments`, `result`, `latency_seconds`) |
| `analysis` | `contextCorruption` (`detected`, `corruptedTurns[]`, `unmatchedTurns[]`), `promptTokenStats`, `initialContextCost` (`systemPromptTokens`, `toolDefinitionsTokens`, `toolCount`), `turnTokenBreakdown[]` |
| `config` | `context[]` — **the actual LLM message context**; `sessionState`; `callMetadata` (`registered_tools`, `state_configurations`, `state_load_errors`, `preload_status`, `transfer_data`, `provider`, `agent_mapping`); `ttsCacheStats`; `usageStats` |

## Mining the logs — the richest surface, and the one that needs a technique

Logs are the **best evidence for a config, startup or run-time error**, because they carry the pipeline's
own reasoning: which services were constructed, what the LLM was actually sent, why a call terminated.
The transcript tells you what was said; `toolCallMetrics` tells you what a tool returned; **only the logs
tell you why the call died.** They are also the largest payload here, so they need a technique rather
than a raw fetch.

### `sessionId` is the scoping key — for voice and chat alike

One concept scopes logs on both surfaces: **the session id**. What differs is only where you read it from
and how much the endpoint does for you.

| | **voice** `/api/call/<callId>/logs` | **chat** `/api/chat/session/<sessionId>/logs` |
|---|---|---|
| the session id is | the call's **`requestId`** | the chat **`sessionId`** (also in the path) |
| scoping | **you pass** `session_id=<requestId>` | **automatic** from the path |
| `start` + `end` | **required** — `400` without them | **optional** — defaults to the session's own window |
| criteria gate | `400 "No search criteria"` unless `session_id`/`filter`/`q` | none — the path id is the criteria |
| adding `filter=` | **AND**-ed with the scope ✓ | ⚠ **REPLACES the scope** — see below |

For voice, the call's `requestId` **is** its session id — the logs prove it:
`workspace.save: Saved session <requestId>` and
`session_tracker: Session unregistered: ws_<requestId>`. So `session_id=<requestId>` is not a workaround,
it is the same key the chat surface names directly.

Chat logs also come from the **same stream** (`default_labels: service_name=voice-ai`) and the same
config-logging modules (`pipeline_factories`, `worker_builder`, `llm_tool_builder`, `call_manager`), so
**every technique below applies to chat unchanged** — including the config mining.

### ⚠ On the chat endpoint, a `filter` silently unscopes the query

The chat route passes `session_id=None if line_filters else session_id` — so the moment you add **any**
`filter=`, the session scoping is dropped and you get everything in the window. Measured on one session:

| Query | Rows | Belonging to that session |
|---|---|---|
| no params | 841 | 837 |
| `filter=\|=:.` | **4239** | 837 — **3402 rows are other sessions** |
| `filter=\|=:<sessionId>` | 841 | 837 ✓ |

**So on chat, if you add a filter, put the session id in the filter chain yourself:**
`?filter=|=:<sessionId>&filter=|~:ERROR|ALERT`. `level=` and `q=` do **not** unscope — only line filters
do. The voice endpoint is not affected; there `filter` is additive.

### ⚠ First get the window right — `createdAt` is NOT the call start

Verified live: a call with `createdAt: 06:07:56` and `duration: 39.78` has log activity spanning
**06:07:14 → 06:08:34** (80 s). `createdAt` sits **42 s after** the first log line, and the pipeline's
**config load happens at 06:07:14** — so a window of `[createdAt − 30s, …]` starts at `06:07:26` and
**misses the entire config/startup phase**. (Call Analyzer hedges about this itself: the playback contract
reports `anchorSource: "created_at_minus_duration"` with a `heuristic-transcript-anchor` warning.)

**Use a generous window, backwards-weighted** — `level=` keeps it cheap anyway:

```
start = createdAt − duration − 90s        end = createdAt + 90s
```

(Only the **voice** endpoint needs this. On chat, omit `start`/`end` entirely and let the route derive the
session's own window — it already applies a buffer either side.)

### The escalation ladder — narrow first, widen by level (measured on one 40-second call)

| Query | Rows | Size | What you get |
|---|---|---|---|
| `&level=error` | **3** | **2.3 KB** | **start here** — the failure itself |
| `&level=warning` | 4 | 2.8 KB | the `[ALERT]` trail leading into it |
| `&level=info` | 21 | 13 KB | runtime narrative: `Transcript [role]:` turns, latency, cache, Kafka, session lifecycle |
| `&level=debug` + a `filter` | 1–11 | 3–61 KB | **the configs** — see below |
| `&level=debug` alone | 1488 | **985 KB** | ⚠ the firehose. Never do this |
| `&q=Transcript` | 12 | 7 KB | the conversation as the pipeline logged it |
| `&filter=!=:unified_serde` | 493 | 309 KB | barely helps — serialization is a third of all lines |

**All four levels work** (`error`, `warning`, `info`, `debug`) even though `/logs/labels/level/values`
only lists `error` and `info` — another reason not to trust that endpoint. Line filters take `|=`
contains, `!=` excludes, `|~` regex, `!~` negated regex (plus case-insensitive `|=i` / `!=i`) as
`filter=<op>:<value>`, repeatable for AND; `q=<text>` is a plain contains. On the voice endpoint all of
these need `start`+`end` and an explicit `session_id=<requestId>`; on chat both are implicit — but mind the
unscoping trap above.

### The configs are at DEBUG — target them, never crawl for them

This is the answer to "what config did this call actually run with": the pipeline logs its resolved
configuration at startup, at DEBUG. Ask for it by name and it is cheap.

| What you want | Query | Cost |
|---|---|---|
| Worker config as fetched **and** as composed | `level=debug&filter=\|~:Raw Config\|Worker Config\|Request Config` | 3 rows / 61 KB |
| **The system prompt actually sent to the LLM** | `level=debug&filter=\|~:system instruction` | 1 row / 4.7 KB |
| **Which tool families were wired — or weren't** | `level=debug&filter=\|~:tools configured\|No .* configured` | 5 rows / 2.7 KB |
| A dangling tool reference, end to end | `level=debug&filter=\|~:not registered` | 11 rows / 25 KB |

What those lines contain, verbatim from a live call:

- `worker_parser.resolve: Raw Config: {…}` — the worker JSON as returned by the AI Worker API, including
  `llm_config` (provider, `base_url`), `greeting_instructions`, and a top-level `instructions: null`.
- `worker_builder.build: Worker Config: {…}` — the **resolved** config: `worker_id`, `initial_agent_id`,
  `goal`, and the **fully composed `instructions`** (your prompt wrapped in `<tts_text_guidelines>` and
  the rest of the platform scaffolding).
- `base_llm: Using system instruction: …` — the composed system prompt as handed to the model. Compare
  this against what you *think* you wrote; it is the ground truth for any prompt-related defect.
- `pipeline_factories.setup_*_tools:` — an explicit **inventory of what was not wired**:
  `No session-state builtin tools configured`, `No A2A agents configured`,
  `No custom code tools configured`, `No skills configured; skipping skills tool`.
  **⚠ This is the earliest possible detection of a dangling `[tool:…]` reference** — it is logged at call
  setup, ~10 seconds before the LLM tries the call and fails. If your prompt names a tool and these lines
  say nothing was configured, you have found the defect before the conversation even started.
- `worker_builder._build_workspace_config:` — auto-injected channel variables, e.g.
  `agent_gender CHANNEL variable: female`, `language CHANNEL variable: default=en, allowed=['en']`.
- `worker_parser._validate_instructions` / `_validate_handoff_config` — prompt and agent-topology
  validation verdicts (`Validated single-agent config: orchestrator + 1 members`).

⚠ **Not every scary DEBUG line is a fault.** Verified live:
`get_session_info ATTEMPT 1 EMPTY | status=404, error={"success":false,"error":"Session not found: …"}`
is immediately followed by `SUCCESS (empty) | reason=no_messages` — a benign first-call miss on a new
session. Read the resolution line before reporting a 404.

### ⚠ The `/logs/labels` trap — do not scope by label

`/logs/labels` looks like the way to narrow, and it is **not**. Two verified failure modes:

1. **`label=` alone returns `400 "No search criteria"`.** Label filters don't satisfy the criteria gate —
   you must *also* pass `session_id`, `filter` or `q`. So a label can never replace them, only add to them.
2. **The labels list is cluster-wide, not call-scoped, so a plausible label silently matches nothing.**
   `request_id` is in the list, and scoping by it returned **0 rows** for a call that has 763. Its
   `/values` holds **934 UUID-format ids belonging to other services** — voice-ai carries its request id
   in the log *line*, not as an indexed label. `default_labels` reveals the real stream selector:
   `service_name = voice-ai`.

**Conclusion: `session_id=<requestId>` (a line-contains match) is the correct and only reliable way to
scope voice-ai logs to one call.** Treat `/logs/labels` as diagnostic curiosity, not a scoping tool.

### What only the logs will tell you

- **Why the call actually ended.** A `stopReason: pipeline_error` decomposes into a readable chain:
  `[ALERT] Silent LLM response from CommotionLLMService#0` → `[ALERT] No fallback services available` →
  `ErrorFrame(error: LLM produced empty response (no content, no tool calls), fatal: True)` →
  `PIPELINE ERROR … terminating call`. **That middle line is a worker config gap, not a platform fault** —
  the model returned empty and there was **no fallback model configured to catch it**, so an
  otherwise-recoverable blip became fatal. Fix: configure a fallback (`control-and-reliability.md`).
- **Exactly what the model was sent**, including the tool definitions — `llm_log_observer` logs each
  `LLM CONTEXT FRAME`, and `base_llm` logs `Generating chat from context [...]`. This is the definitive
  proof for a dangling `[tool:…]` reference or a context that lost a turn.
- **Recording/audio gaps in plain words**, e.g. `Recording skipped: AudioBufferProcessor flushed with
  empty buffers … No audio captured`.
- **Session lifecycle**, e.g. `Session unregistered: ws_<requestId> (type=websocket, duration=42.0s)` —
  note the `ws_` prefix on the session id.
- Useful modules to filter on: `pipeline_events`, `llm_switcher`, `llm_log_observer`, `call_manager`
  (logs `Transcript [role]:` lines), `call_state_tracker`, `session_tracker`, `metrics_manager`.
  The noisiest by far are `unified_serde` and `text_transformer` — pure serialization chatter.

⚠ **A call's log lines survive the worker being deleted** (verified — Loki and the call document are
independent of the worker record), so a post-mortem stays possible after cleanup.

## Chat endpoints

| Path | Params | What it gives you |
|---|---|---|
| `/api/chat/sessions` | `sessionId`, `workerId`, `agentId`, `sessionType`, `sort`, `page`, `limit` (≤50) | the session list |
| `/api/chat/session/<id>` | `type` | ~32 KB: `header`, `runs`, `config`, `configTurns`, `performance{kpis, perTurn, modelBreakdown}`, **`errors[]`**, `sessionState`, `agentSystems` (resolved per-agent prompts) |
| `/api/chat/session/<id>/logs` (+ `labels`, `labels/<name>/values`) | `level`, `q`, `filter`, `label`, `direction`; `start`/`end` **optional** | the pipeline trace — **auto-scoped + auto-windowed**; ⚠ a `filter` unscopes it (see Mining the logs) |
| `/api/chat/config` | `type=worker\|agent\|workflow`, `id` | ⚠ ~283 KB — the session detail already carries this |

For copilot sessions the chat `sessionId` **is** the voice `requestId`, so the two surfaces join there.

## AMD (answering-machine detection)

`/api/amd-sessions` · `/api/amd-session/<request-id>` · `/api/amd-session/<request-id>/raw-inbound/summary`.
RCA-only — there is no simulation path for AMD, so the Phase 3/5 gates cannot close on it.

## Not usable from this tool

Audio and file routes return bytes, not JSON: `/api/call/<id>/recording`,
`/api/call/<id>/supervisor/recording`, `/api/call/<id>/stt-segment-audio/<turn>/<kind>`,
`/api/call/<id>/raw-inbound/reconstructed-recording`, `/api/recording/<file>[/user-channel|/bot-channel]`,
`/api/export`, `/exports/download`. Don't call them; if the user needs to *listen*, point them at the
Call Analyzer UI.

## Gotchas (all verified live)

- **⚠ `fields=all` does not work.** The docstring claims it does; the code does exact set membership on
  the comma-split value, so `all` matches no section and you silently get the shell. **Name each
  section.**
- **⚠ `workerId` on a call is `<aiWorkerId>_<version>`** — e.g. `6a64c0f475e04e98f9cf01f3_1`. Strip the
  `_N` suffix for the `aiWorkerId` used on the BE plane; `N` is the version that call ran on. Both the
  suffixed and bare forms occur in call records (7 vs 2 calls for the same worker), so **query both
  forms** when counting a worker's calls, or you will undercount.
- **⚠ `timeWindow` accepts only `5m`, `15m`, `1h`.** Anything else — `24h`, `7d` — is **silently
  ignored** and you get unfiltered results that look like a filtered set. `date` filters a single UTC
  day (`YYYY-MM-DD`). There is no date-range filter on `/api/calls`.
- **⚠ `/logs` needs `start` **and** `end`** (ISO-8601) or it `400`s, plus at least one of `session_id` /
  `filter` / `q` — **`label=` does not count** and returns `400 "No search criteria"` on its own. And
  **`session_id` is a line-*contains* filter, not a label match** — a short or wrong value matches nearly
  every line in the window. Use the call's `requestId`, widen the window **backwards** past `createdAt`
  (which is not the call start), and start with **`level=error`** (3 rows / 2.3 KB vs 1488 / 985 KB on a
  real call). Loki's row limit is hardcoded 5000 server-side and cannot be lowered from the query — the
  tool's byte cap is the backstop, not your plan. Full technique in **Mining the logs** above.
- **⚠ `createdAt` is not the call start** — it sat 42 s *after* the first log line on a verified call, and
  `duration` (39.8 s) was half the real activity span (80 s). Anything that keys a time window off
  `createdAt` alone will miss the startup/config phase.
- **`/api/filter-options` returns empty lists by design.** It was gutted for performance; don't read it
  as "this worker has no calls".
- **`requestId` is the join key from a simulation back to its call — and it is load-bearing** (verified
  live). A **scenario-run `id` IS the call's `requestId`**, so
  `/api/calls?requestId=<scenario-run-id>` resolves any run to its call in one hop. This is what lets you
  judge a run from its transcript when the evaluator returns no verdict — which is the normal case, see
  `repro-and-gates.md` §1a. Learn this one; you will use it on every verify pass.
- **`callId` (`call_6a47ae1895ff`) is the join key to the eval domain's `voiceInteractionId`.** Getting
  from it to a BE `conversationId` (what `POST /scenario/generate-from-conversation` needs) goes through
  `GET /conversation/worker-conversations?workerId=…` — see `repro-and-gates.md`. **This link is the one
  step not yet confirmed live**; the repro phase has a fallback for exactly that reason.
- **`callMode` tells you what kind of run you are looking at**: `SIMULATION` (a scenario-run),
  `COPILOT` (a `POST /aiworker/run` text spot-check), `OUTBOUND` / `INBOUND` (real traffic). A
  `COPILOT` call has no caller audio by design — don't diagnose it as a mute-caller artifact.
- **`registered_tools` always includes the built-ins** (`end_call` at minimum), so
  `initialContextCost.toolCount: 1` means "one tool, and it's built-in" — not "a custom tool is wired".
  Read the names. A `[tool:<name>]` in the prompt with no matching entry is a dangling reference, and at
  runtime it surfaces as `toolCallMetrics[].result == "Error: function '<name>' is not registered."`
- **Skills have their own dangling-reference check, and it is the skill analogue of `registered_tools`.**
  `/api/chat/session/<id>` → `config.skills[]` lists every skill the agent advertised, each with
  `name`, `description`, `rawInstructions`, **`isReferenced`** and **`referencedBy[]`**. Two distinct
  failures to look for: a `[skill:<name>]` in the prompt with **no matching entry** in `config.skills[]`
  (bound id missing — the agent had no such skill to read, and will usually answer from general
  knowledge instead of admitting it), and an entry with **`isReferenced: false`** — bound but never
  pointed at from the prompt, which is a half-finished attachment: add the `[skill:<name>]` reference
  to the agent's `instructions` so the routing is explicit and reviewable.
- **⚠ A custom tool is listed under an opaque MCP-server hash, not its action name.** In
  `/api/chat/session/<id>` → `config.tools[]`, a custom/connector tool appears as
  `{name: "27bcd3ab9369455887171300d7f54cbf", type: "mcp"}` — searching that list for
  `check-slot-availability-…` finds nothing. The action names live **inside** each entry, under the
  `functions_vs_metadata` field (and `tool_data.included_actions`). Read those before concluding a
  tool isn't wired.
- **Every mention token resolves to a tool call, so the trace is where you prove any of them fired**
  (verified live 2026-08-10): `[knowledge:…|id:…]` → `search_knowledge_base` with
  `filters:[{key:"id", value:<the id>}]`; `[skill:…]` → `read_skill`; `[tool:…]` → the action itself.
  A turn that answered a knowledge-backed question with **no `search_knowledge_base` call** was not
  grounded, whatever the reply text implies — and an empty `references[]` on the run means the search
  matched nothing (usually a wrong or stale id in the token).
- **Skill loading is visible as a tool call.** The resolved system prompt carries an
  `<available_skills>` manifest of names + descriptions only; the body arrives via the built-in
  **`read_skill(name)`**, which shows up in `runs[].tools` / `steps` with the full instructions as its
  result. If the agent answered a question a skill covers **without** a `read_skill` call in that turn,
  the answer was not grounded in the skill — fix the skill's `description`, not its body.
- **`transcription[]` contains `state_snapshot` rows**, not just `user`/`assistant` turns — they record
  state-variable changes mid-call and are useful, but don't count them as conversation turns.
- **Access is role-scoped to the connected user** (not blanket-admin). An `admin` sees unmasked
  transcripts and internal detail across every workspace; `analyst`/`reader` see a reduced, redacted,
  and workspace-scoped view. Regardless of role, treat what you fetch as sensitive: quote only what the
  RCA needs, and never echo a whole transcript of a call the user didn't ask about. If your reads come
  back masked/partial or an endpoint `403`s, that is your role — say so rather than guessing at the
  hidden content.
