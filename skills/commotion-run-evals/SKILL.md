---
name: commotion-run-evals
description: >-
  Run a Commotion worker against its test scenarios as an automated simulation and report the
  eval score — define eval metrics (optional), launch the simulation for a worker/version, poll it to
  completion, and read the pass-rate plus a per-scenario pass/fail breakdown with failure reasons. Use
  this whenever the user wants to run evals, run a simulation, test a worker against scenarios, get a
  pass-rate / eval score, or "see how the worker does" — e.g. "run my scenarios and tell me the pass
  rate", "evaluate the renewal bot". This is step 3 of the quality loop (create-worker →
  generate-scenarios → **run-evals** → improve-worker). Needs scenarios to exist first
  (commotion-generate-scenarios). Calls the Commotion backend through the thin Commotion MCP server
  (OAuth — no API key in the transcript).
allowed-tools: Read, AskUserQuestion, mcp__commotion-dev3__commotion_request, mcp__commotion-tcuat__commotion_request, mcp__commotion-tcprod__commotion_request, mcp__commotion-dev3__commotion_schema, mcp__commotion-tcuat__commotion_schema, mcp__commotion-tcprod__commotion_schema, mcp__commotion-dev3__commotion_analyzer, mcp__commotion-tcuat__commotion_analyzer, mcp__commotion-tcprod__commotion_analyzer
---

# Commotion: Run Evals (Simulate + Score)

Run a worker's scenarios as a **simulation** and get back the **eval score** — the pass-rate (the
percentage, 0–100, of scenarios whose goal the worker achieved), plus a per-scenario breakdown with the reason each
failure failed. Optionally define **eval metrics** (Hallucination, CSAT, latency, custom domain rules)
for richer signal. You make the platform I/O through the connected **Commotion MCP** server's two tools
(`commotion_request` / `commotion_schema`). **Every write (metrics, the run) is shown to the user and approved first.**

**Auth is automatic (browser login).** The Commotion MCP handles auth via OAuth: on first use it
opens a Commotion login in the browser, then attaches the user's token to every `commotion_request` /
`commotion_schema` call for you — the token never enters the conversation. Never ask the user for an
email/password or a token, and don't pass a `token` argument. If the two tools aren't available at
all, the MCP isn't connected — ask the user to add/authorize it via `/mcp`.

This is **step 3 of the worker quality loop**:

```
create-worker → generate-scenarios → [run-evals] → improve-worker
                                         └──────── repeat until pass-rate ≥ threshold ────────┘
```

Its inputs are the scenarios from `commotion-generate-scenarios` (step 2); its output — the pass-rate
and the failing-scenario analysis — feeds `commotion-improve-worker` (step 4), which uses it to decide
whether to keep iterating. The headline **eval score is `SimulationResponse.passRate`** — a
**percentage (0–100)**, not a 0–1 fraction (verified live: an all-pass run returns `100.0`).

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

## Prerequisites (verified live — every channel simulates; you need a live runtime)

1. **Simulations work for VOICE, CHAT *and* STRUCTURED-OUTPUT workers — always use `/simulation/run`,
   never a live session.** There is **one** simulation endpoint for every channel; the channel comes
   from each **scenario's `aiAgentChannelType`** (`VOICE` or `CHAT`), not from a different endpoint.
   Verified live 2026-08-03: a chat worker returned `passRate 100.0` and a `STRUCTURED_OUTPUT` worker
   returned `scenarioEvaluationResult: "PASS"`, both through `POST /simulation/run`, which echoes
   **`onlyChatScenarios: true`** for an all-chat batch. `GET /scenario/dropdown-config` lists
   `channelType: [VOICE, CHAT]`.

   > ⚠ **A `STRUCTURED_OUTPUT` agent is on the CHAT channel** — the schema says `aiAgentChannelType`
   > *"Must be CHAT for CHAT_AGENT / STRUCTURED_OUTPUT and VOICE for VOICE_AGENT"*, and a freshly
   > provisioned SO agent reports `aiAgentChannelType: "Chat"`. So SO scenarios take
   > `aiAgentChannelType: "CHAT"`.

   **Never substitute a live run for a simulation.** `POST /aiworker/run` and ad-hoc chat sessions are
   **not** simulations: Call Analyzer tags them `callMode: COPILOT` and they land under Observability's
   **live** filter, produce no `scenarioEvaluationResult`, no `passRate`, and nothing under
   **Simulations → Runs**. If you catch yourself inventing a session id to "test" a chat worker, stop —
   create a scenario and run `/simulation/run` instead.

   **Personality voice rule is channel-specific.** A **voice** scenario needs a `voiceEnabled: true`
   personality with a voice, or the caller never speaks. A **chat/SO** scenario needs the opposite —
   `voiceEnabled: false` is correct and sufficient (verified: the passing chat + SO runs above both used
   a `voiceEnabled: false` personality).

2. **The worker version must have a running runtime — i.e. the worker must have been deployed at
   least once.** A worker that was **never deployed** returns *"Worker is not available for worker
   Id: …"* (from `/aiworker/run`) and its sims/AI-generation fail. Once the worker has a live version,
   a **draft version of it CAN be simulated** (verified: draft v1 simulated while v0 was live) — which
   is what makes the draft-only improve loop work. So: deploy once, then iterate+simulate on drafts.

### ⚠ Don't read a generic simulation error as "this channel isn't supported"

A failing run reports only *"An error has occurred during simulation. Please contact support with
reference number …"* — with `duration 0.0` and `scenarioRunStatusLabel: "Simulation Error"`. That
message says nothing about the channel. Verified live 2026-08-03: on the worker that produced it, the
simulation had in fact started correctly and injected the scenario's `userScript` verbatim; the real
cause was in the session's own error list — a `litellm.AuthenticationError: Incorrect API key
provided` naming the model the pipeline actually called,

a **worker LLM credential fault**, not a channel limitation. ⚠ **That error text names the model by its
internal upstream identifier, not its Commotion name — paraphrase it, never paste it to the user** (see
"Never surface Commotion's upstream vendors" below). Before concluding anything from a generic
simulation error, read the session: `commotion_analyzer { "path": "/api/chat/session/<scenario-run-id>" }`
for a chat/SO run (`errors[]` + `runs[].error`), or `/api/calls?requestId=<scenario-run-id>` for voice.
An earlier revision of this skill mis-generalised exactly this error into a false "voice-only" rule.

### ⚠ Never surface Commotion's upstream vendors

Commotion's first-party models are served under Commotion names, but the **Call Analyzer plane and the
pipeline logs report the internal upstream identifiers instead** — the plane swaps the vendor in for the
`commotion` prefix, so `commotion-3.6-35b` arrives as `qwen3.6-35b`, `commotion-4-31b-it` as
`gemma-4-31b-it`, `commotion-omni` as `qwen3-omni`, plus `openai/qwen3-next` inside litellm errors and
`krisp` for noise cancellation. Repeating any of them to a customer discloses Commotion's supply chain.
**Map every model/provider name back to its Commotion-facing name before it reaches the user**, in
prose and in quoted errors alike; the `commotion-*` codes on the BE plane are already customer-facing
and safe. **BYOK is the opposite** — when the customer brought the key (`isThirdParty: true`, a real
`credentialId`; e.g. their Eleven Labs personality voice or an Azure OpenAI fallback), name the
provider plainly. Full mapping table and the tell-them-apart test:
`commotion-debug/references/call-analyzer-api.md` ("Never surface Commotion's upstream vendors").

## When to use this

The user wants to actually run the test set and get a score, or asks to "simulate" / "evaluate" /
"test" a worker. Scenarios must already exist for the worker (run `commotion-generate-scenarios`
first). To then improve the worker until it clears a threshold, that's `commotion-improve-worker`.
**For the whole build → test → improve pipeline in one request, use the `commotion-quality-loop`
orchestrator** (it invokes this skill as its eval step).

## How this skill talks to the platform (read first)

All platform I/O goes through the connected **Commotion MCP** server — two tools, no scripts, no keys.
It's one unified backend (scenario/simulation/eval endpoints share the spec with `/aiworker`), so this
is the **same transport as `commotion-create-worker`**:

- **`commotion_request`** — one authenticated call: `{ "method": "GET|POST|PUT|DELETE", "path": "/…",
  "body": <JSON, for writes> }` → `{ "status", "body" }` (a non-2xx is **returned, not thrown** —
  read it and adjust). Pass a **path** (the base URL is fixed server-side) and the request payload as
  `body` — no temp files.
- **`commotion_schema`** — a bundled request schema: `{ "schema_name": "RunScenariosRequest" }` → the
  JSON Schema with its `$defs`. Any component name in the live spec works. **Never invent a field that
  isn't in the schema.**
- **`commotion_analyzer`** — one **GET** against the **Call Analyzer** plane: `{ "path": "/api/…" }` →
  the same `{ "status", "body" }` shape. This is where the *actual calls* live. **A simulation is just a
  real call with a robot caller**, so every scenario-run has a full transcript, per-turn latency, tool
  calls with their results, and audio metrics waiting here — far richer than `evaluationReasoning` alone.
  Use it in Phase 3 to confirm the runs really happened and in Phase 4 to explain *why* they failed.
  ⚠ **Optional**: the MCP registers it only in authenticated mode, and even then the connected user
  may lack access (`{ "status": 403 }`) or see a role-reduced view. On any of those — absent, `403`,
  or masked/partial — say so once and fall back to `evaluationReasoning`; never block a run on it.

**Auth is automatic — there is no key.** The MCP client owns OAuth: the first time the
Commotion MCP is used it opens a Commotion login in the browser, then attaches the user's token to
every call. Never ask the user for a token. If the two tools aren't available, the Commotion MCP
isn't connected — ask the user to add/authorize it (in Claude Code: `/mcp` → **commotion** →
Authenticate), then continue.

**Read ids from results, not `jq`:** `commotion_request` returns the parsed `body` — after `POST
/simulation/run` the run id is `body.id`; from a list it's `body[0].id`. Feed that id into the next
call's `path`.

The endpoint map is in `commotion-generate-scenarios/references/eval-domain-api.md` (the canonical
map). Eval-metric design is in `references/eval-metrics.md`; the run lifecycle + how to read scores is
in `references/simulation-and-results.md`. The Call Analyzer endpoint map — what each `?fields=` section
returns, payload sizes, and the gotchas — is
`commotion-debug/references/call-analyzer-api.md` (canonical for that plane; read it before your first
`commotion_analyzer` call).

**Execution rules:** one phase at a time; read the reference a phase names before acting; show every
write before making it; **never start a run while another is active** (Phase 2).

## Phase 0 — Establish the target, then ground in the real schema

First fix what you're testing: the **worker id + version** to evaluate (`<worker-id>` — in the loop,
the **draft** under improvement) and the **scenario ids** to run (from `commotion-generate-scenarios`
— `commotion_request` `{ "method": "GET", "path": "/scenario?aiWorkerId=<worker-id>" }`, reading each
record's `version` from `body`). Then ground in the schema:

1. `commotion_schema` `{ "schema_name": "RunScenariosRequest" }` and `{ "schema_name": "EvalMetricRequest" }`.
2. `commotion_request` `{ "method": "GET", "path": "/eval-metric?aiWorkerId=<worker-id>" }` → metrics
   already on the worker (don't re-create duplicates).
3. `commotion_request` `{ "method": "GET", "path": "/scenario/dropdown-config" }` → `maxScenarioRunLimit`
   (cap on runs-per-scenario × scenarios) — respect it.
4. `commotion_request` `{ "method": "GET", "path": "/aimodel?pageSize=200" }` → provider/model codes for
   the **simulator LLM**. ⚠ **Pass `pageSize`** — the endpoint paginates at ~10 rows and the bare call
   returns 10 junk `azure_openai` test rows with **zero Commotion models** (verified live 2026-08-06).
   The simulator `llm` must stay on a `commotion` provider, so you need the paged list to find one.

## Two evaluation surfaces (read before Phase 1 — verified live)

The platform has **two separate things** both loosely called "evals," and they populate different UI
tabs. Don't conflate them:

1. **Simulation scenario pass/fail** — from `POST /simulation/run` → `SimulationResponse.passRate` +
   per-scenario `scenarioEvaluationResult` (PASS/FAIL against each scenario's **goal**). Shown under
   **Simulations → Runs**. **This is the loop's gate** and needs no metrics.
2. **The Evals dashboard** (Evals → Overview/Metrics/Alerts: *Total calls evaluated, Metrics Evaluated,
   Pass Rate, quality*) — driven by **eval-metrics** (`/eval-metric` + `/eval-result`). It stays **empty
   (0 / No Data)** until you (a) create eval-metrics with `simulation:true` and (b) run a simulation so
   they score the calls. `SimulationResponse.avgQuality` is also `null` until metrics exist.

So: scenario pass-rate ≠ Evals-dashboard data. If the user asks "why is the Evals dashboard empty?",
the answer is almost always "no eval-metrics defined." Phase 1 fills it.

## Phase 1 — Define eval metrics (optional for the pass-rate gate; REQUIRED to populate the Evals dashboard)

Skip if the user only wants the scenario pass-rate. Do this when they want metric-level quality
(Hallucination, CSAT, tone, compliance) or the Evals dashboard populated. Design from the use case.

You can create **both standard (predefined) and custom** metrics — but the two bodies differ sharply,
and standard has a trap (verified live):

- **Standard / predefined → POST a minimal shell, then PUT to hydrate.** A standard metric can't be
  created hydrated in one call: a full-body POST 500s (poison field: `name`), and a minimal POST
  (`{standardEvalMetricId, aiWorkerId, version, simulation, observability}`) returns 200 but a **hollow
  shell** (blank name/type — which can also break the eval pass). So POST the shell, then
  `PUT /eval-metric/{id}` the full definition (mirrored from the catalog) **dropping any empty-string
  field — especially `evaluationMethod: ""`** (empty `evaluationMethod` 500s the PUT; CORE_PLATFORM
  metrics like CSAT/Sentiment have it empty, so omit it). Fetch the catalog + valid ids with
  `GET /eval-metric?metricSourceType=STANDARD` (hallucination, relevancy, response_consistency,
  csat_score, sentiment, appropriate_call_termination, latency, tool_call_success_rate,
  conversation_progression, …). Include NUMERIC/RATING ones (hallucination, csat_score, relevancy) for
  richer quality signal. Full recipe in `references/eval-metrics.md`.
- **Custom → FULL body.** For domain rules the catalog doesn't cover, supply everything:
  ```jsonc
  { "name":"No Professional Advice", "metricSourceType":"CUSTOM", "outputType":"BOOLEAN",
    "thresholdCondition":"EQ", "thresholdValue":"true", "enumValues":[],
    "evaluationCriteria":"Pass if the agent never gives legal/financial/medical/tax advice…",
    "evaluatorType":"AI_PLATFORM", "evaluationMethod":"LLM_JUDGE", "category":"Compliance",
    "simulation":true, "observability":true, "aiWorkerId":"<id>", "version":<LIVE> }
  ```

Shared rules (verified): `version` must be the worker's **LIVE** version (a never-deployed draft
version and `null` both 500 — deploy first, create metrics at the live version); `simulation:true`
scores simulated calls (`observability:true` for live). `outputType` ∈ BOOLEAN/NUMERIC/RATING/ENUM;
`thresholdCondition` ∈ EQ/LTE/GTE; `thresholdValue` is a **string** ("true"/"10"/"70"). The list body
can contain raw newlines (parse tolerantly). Show each before writing. Full detail + catalog in
`references/eval-metrics.md`.

> The loop's **gate is the scenario pass-rate** (surface 1). Metrics (surface 2) add quality signal +
> the dashboard; they are not required for the pass-rate gate.

## Phase 2 — Select scenarios and run the simulation  ·  HUMAN INPUT REQUIRED

1. **Check nothing is already running** — `commotion_request` `{ "method": "GET", "path":
   "/scenario-run/active?aiWorkerId=<worker-id>" }` → if `body` is `true`, wait (the platform runs
   simulations **sequentially**; starting a second is blocked).
2. **Choose scenarios + runs-per-scenario.** `RunScenariosRequest.scenarioIdToRunPerScenarioMap` maps
   each `scenarioId` → how many times to run it (run a scenario several times to test consistency).
   Keep total runs within `maxScenarioRunLimit`.
3. **⚠ Cap each simulation at 4 total scenario-runs, and batch anything larger (verified operationally).**
   A single `/simulation/run` runs its scenario-runs concurrently over websockets; more than **4** at
   once exhausts the websocket connections and runs start **failing with websocket connection errors**
   (they come back `COMPLETED`-instantly with `passRate 0.0`/`avgLatency null` — the failure signature).
   So keep **`sum(scenarioIdToRunPerScenarioMap.values()) ≤ 4` per simulation**. If the user picked more
   scenarios/runs than that, **split into sequential batches of ≤4**: run one batch, poll it to
   completion (Phase 3), then start the next (each batch is a fresh `/simulation/run` — check
   `/scenario-run/active` is clear first), and aggregate the pass-rates across batches at the end.
4. **Confirm and run** — show the user which scenarios, how many runs each (and, if batching, the batch
   plan), and the simulator `llm`; on yes, call `commotion_request` `{ "method": "POST", "path":
   "/simulation/run", "body": {aiWorkerId, version, scenarioIdToRunPerScenarioMap:{<scenarioId>:<nRuns>,…}
   (≤4 total), maxDuration, maxTurns, llm:{provider,model}} }` and read the run id from the result:
   **`<sim-id> = body.id`** (reused in every later call). **Run against the version under test** (the
   draft, in the loop).

## Phase 3 — Poll to completion and read the score

Repeatedly call `commotion_request` `{ "method": "GET", "path": "/simulation/<sim-id>" }` and read the
headline numbers from `body` — `passRate`, `passCount` / `totalScenarios`, `avgQuality`,
`avgLatency(InMillis)`, `completedScenarios` — until the run is genuinely terminal.

> ## ⏱ How long this takes, and where the duration comes from (verified live 2026-08-06)
>
> **Budget 2–7 minutes for a ≤4-run voice batch, and expect wide variance.** Measured end-to-end on
> four live voice batches:
>
> | Batch | Per-run durations | `timeToComplete` |
> |---|---|---|
> | 2 runs | 53.0, 75.5 s | **75.6 s** |
> | 2 runs | 52.0, 71.8 s | **69.5 s** |
> | 4 runs | 62.8, 63.4, 63.5, 64.5 s | **101.9 s** |
> | 4 runs — *same worker, same scenario, same 4 runs as above* | ~54 s each | **374.2 s (6.2 min)** |
>
> ⚠ **The last two rows are the same batch shape and they differed by 3.7×.** Per-run conversation
> durations barely moved (~54–64 s each); the queueing and finalisation around them is what varies. So
> **elapsed time alone never tells you a batch is broken** — a 6-minute ≤4-run batch is within normal
> range and that one finished with `passRate 100.0`. Chat/SO batches are faster.
>
> Voice runs execute **concurrently**, so a batch costs far less than the sum of its runs, but it is
> **not** simply the longest run either — there is queueing overhead on top, and that overhead is the
> variable part.
>
> **Never narrate your own elapsed time — the server reports the duration, so read it.** Three fields
> on `SimulationResponse` carry timing, and only one of them is trustworthy:
>
> | Field | Unit | Verified behaviour |
> |---|---|---|
> | `timeToComplete` | **milliseconds** | the batch's end-to-end duration — `75593` = 75.6 s. ✅ **This is the number to report.** |
> | `executionTime` | seconds | ⚠ an aggregate over per-run durations, **unreliable** — see below |
> | `totalExecutionTime` | seconds | ⚠ same aggregate, **unreliable** — see below |
>
> ⚠ **`executionTime` / `totalExecutionTime` silently under-count when a run errors.** Measured on two
> batches of the same two scenarios: batch A had per-run durations `53.04 + 75.49` and reported
> `executionTime 64.27` (their mean) / `totalExecutionTime 128.53` (their sum) — but batch B had
> `51.99 + 71.81` and reported **`51.99` for both**, counting only the run that passed. The aggregate is
> computed from whichever runs had posted a duration at aggregation time, so an errored run may or may
> not be included. **Don't derive a duration from them, and don't present them to the user.** Use
> `timeToComplete`; for per-run seconds read each `ScenarioRunResponse.duration` directly.
>
> A non-null `executionTime` on a still-`PENDING` sim does at least confirm **live progress** — that
> much is safe to use as a liveness check while polling.
>
> **Your polling wall-clock is not the simulation's duration.** It includes your own waits, retries and
> turnaround, and it will drift far above the real figure if you poll lazily. Reporting "this took 29
> minutes" off your own clock when `timeToComplete` says 76 seconds is a reporting bug — it has caused
> sessions to be abandoned and users told to "run it in the UI instead" on runs that had already
> finished in under two minutes. **Quote `timeToComplete` (converted to seconds), never your own
> elapsed time.**
>
> **Poll cadence:** first poll at **~45 s**, then every **~30 s**. Don't sleep for minutes between
> polls, and don't hammer it either. A batch can legitimately need a dozen polls — that is normal and
> is **not** a reason to stop.
>
> **⚠ `duration: 0.0` on a run whose `status` is `RUNNING` or `QUEUED` is normal** — the field is only
> populated when the run reaches a terminal state. It is *not* evidence of a stuck or failed run. The
> "near-zero duration = the run never happened" signature in `references/simulation-and-results.md`
> applies **only to terminal runs**. Do not call a batch "stuck" because in-flight runs read `0.0`.
>
> **Judge "stuck" by progress, never by elapsed time.** A ≤4-run batch has been observed taking 6.2
> minutes and finishing at `passRate 100.0`, so a clock alone proves nothing. The liveness signal is
> **`totalExecutionTime` rising and/or `completedScenarios` advancing between polls** — both climb
> steadily while work is happening (observed: `totalExecutionTime` 105.9 → 154.2 → 216.8 across polls
> as `completedScenarios` went 2 → 3 → 4).
>
> Only suspect a genuine stall when **`completedScenarios` AND `totalExecutionTime` are both frozen
> across several consecutive polls spanning 10+ minutes**. Then say so plainly with the numbers and read
> the runs' `failureReason` / session errors — that is a platform fault to report, not a reason to hand
> the task back to the user. **Never end a run-evals task by telling the user to go run it in the UI.**

**Wait for evaluation to finish before reading `passRate` as final.** Each scenario-run passes through
EVALUATION_* states *after* its conversation completes, and `passRate` isn't final until every run is
evaluated — so don't stop the moment the simulation looks done: confirm `completedScenarios ==
totalScenarios` and no runs are still in an EVALUATION_* state (`commotion_request` `{ "method": "GET",
"path": "/scenario-run?simulationId=<sim-id>" }`, checking each record's `status` in `body`) before
trusting the number. (`SimulationResponse.status` is an unconstrained string — read `statusLabel` and
the counts rather than matching a hard-coded token; the **verified terminal value is `COMPLETED`** /
`statusLabel: "Completed"`.) ⚠ **`completedScenarios` counts runs that ended in *any* terminal state,
including `FAILED`** — verified live: a 2-run batch reported `completedScenarios: 2` with one run
`COMPLETED/PASS` and one `FAILED/Simulation Error`. So "all scenarios completed" means "nothing is
still in flight", not "everything ran successfully". See `references/simulation-and-results.md` for
the full status progression.

`SimulationResponse.passRate` **is the eval score — a percentage 0–100** (verified live; an all-pass
run returns `100.0`). Report it with the raw count (e.g. "6 of 10 passed = 60%") plus `avgLatency` and
the duration from `timeToComplete`. (Note: `avgQuality` is often `null` unless custom eval-metrics are
defined — the scenario-goal PASS/FAIL is the signal that drives `passRate`.)

> ⚠ **`passRate`'s denominator is *decided* runs, not `totalScenarios` (verified live 2026-08-06).**
> A 2-run batch where one run passed and one came back `Simulation Error` (no verdict) reported
> **`passRate: 100.0`, `passCount: 1`, `totalScenarios: 2`** — i.e. 1/1 decided, not 1/2. So the
> backend already excludes undecided runs; **don't re-exclude them and don't recompute the percentage
> yourself.** The corollary is the one that matters: a flattering `passRate` can rest on a single
> decided run. **Always report it as "X% (n of N runs decided)"** — "100% (1 of 2 runs decided; 1
> simulation error)" is the honest line, and it is a very different handoff than a bare "100%".

> ## ⚠ Before you report `passRate`, check that the runs were actually evaluated (verified live)
>
> When **no** run gets a verdict there is nothing to divide, and `passRate` collapses to `0.0`. So a
> `0.0` can mean "the worker failed everything" **or** "nothing was scored", and those demand opposite
> responses. Verified live 2026-07-28: across 8 runs on a deliberately broken worker,
> **0 produced a usable verdict** — they came back with `scenarioEvaluationResult` of **`ERROR`** or an
> **empty string** and `scenarioRunStatusLabel` of `Evaluation Error` / `Simulation Error`, on calls that
> had run 40–60 seconds with complete transcripts. One observed cause is an evaluator-side bug:
> *"Goal completion evaluation failed: 3 validation errors for EvaluationMessage … `channel_types.0`
> Input should be 'voice' or 'chat' [input_value='customer']"*.
>
> So always split the denominator in Phase 4 and **report `passRate` alongside how many runs were
> decided** — "0% (0 of 4 runs evaluated — evaluator errors, score not meaningful)" is the honest
> reading, and it is a very different handoff to `commotion-improve-worker` than "0% (4 of 4 failed)".
> Reporting a bare `0.0` from unevaluated runs sends the improve loop chasing a defect that may not exist.
> To recover the real verdicts, read the transcripts — `commotion-debug` does this through Call Analyzer
> (`/api/calls?requestId=<scenario-run-id>` → `?fields=transcript,metrics`); see
> `commotion-debug/references/repro-and-gates.md` §1a.

## Phase 4 — Per-scenario breakdown (the diagnosis fuel)

Call `commotion_request` `{ "method": "GET", "path": "/scenario-run?simulationId=<sim-id>" }` and read
the per-scenario records from `body`. Each `ScenarioRunResponse` gives `status`,
`scenarioRunStatusLabel`, `quality`, `scenarioEvaluationResult`, and — for failures — `failureReason` +
`evaluationReasoning` (plus `failuresReasoning[]` / `passesReasoning[]`).

**Bucket every run into one of three, and never collapse them:**

| Bucket | How to tell | Report as |
|---|---|---|
| **Pass** | `scenarioEvaluationResult == "PASS"` | a pass |
| **Fail** | `scenarioEvaluationResult == "FAIL"` — a real behavioural verdict with `evaluationReasoning` | a fail, with the reason |
| **Not evaluated** | `scenarioEvaluationResult` is **`ERROR`** or **`''`**, or `scenarioRunStatusLabel` is `Evaluation Error` / `Simulation Error` | **excluded from the pass-rate** — say how many and why |

⚠ `status: COMPLETED` does **not** mean a run was evaluated (verified live: `COMPLETED` +
`Evaluation Error` + `ERROR` on a 55-second call). And a mute-caller run — `audioMetrics.userTurnCount: 0`
with `stopReason: user_idle_timeout` — is not a behavioural fail either; it means the personality wasn't
voice-enabled and the scenario never ran (see `commotion-generate-scenarios` Phase 2).

Present a table: scenario name · pass/fail/not-evaluated · why. **This is exactly what
`commotion-improve-worker` consumes** to decide what to fix — so a not-evaluated run must reach it
labelled as such, never as a failure to diagnose.

### Then read the actual calls — `evaluationReasoning` is an opinion, the call is the evidence

Every scenario-run produced a real call, and **the scenario-run `id` is that call's `requestId`**. So for
each failing (or not-evaluated) run:

```
commotion_analyzer { "path": "/api/calls?requestId=<scenario-run-id>&limit=1" }        → the callId
commotion_analyzer { "path": "/api/call/<call-id>?fields=transcript,metrics,analysis" } → ~12 KB
```

Read four things the eval surface cannot tell you, and fold them into the "why" column:

| Question | Where |
|---|---|
| **Did the run actually happen?** `userTurnCount: 0` / `userAudioSeconds: 0.0` = mute caller, the scenario never ran | `audioMetrics` |
| **Did the tools fire, and what came back?** An empty `toolCallMetrics` when the flow needed a tool, or a `result` like `"Error: function 'x' is not registered."`, changes the diagnosis completely | `toolCallMetrics[]` |
| **Was "slow" real, and slow where?** A slow *tool* and a slow *model* need opposite fixes | `latencyMetrics.summary`, `toolCallMetrics[].latency_seconds` |
| **Was this the worker's fault at all?** `contextCorruption.detected`, `llmFallbackEvents`, `state_load_errors` mark platform artifacts | `analysis` section, `config` section |
| **Why did the run end that way?** the only place that answers it | `/logs?session_id=<scenario-run-id>&level=error` — see below |

**For any `stopReason` you can't explain — and for every config or startup fault — read the error logs.**
This is the single highest-value query on the plane and it is cheap when scoped:

```
commotion_analyzer { "path": "/api/call/<call-id>/logs?session_id=<scenario-run-id>&level=error
                              &start=<createdAt − duration − 90s>&end=<createdAt + 90s>" }
```

~3 rows / 2 KB versus 1488 rows / 985 KB unfiltered. Escalate by level only as needed —
`level=error` → `warning` → `info` — and **never `level=debug` without a `filter`**.

⚠ Two things that will otherwise waste the query. **`session_id` is the session id, and for a scenario-run
that is the run's own `id`** — the same value the call carries as `requestId`, which is why the join works
in both directions. (For a chat session use `/api/chat/session/<sessionId>/logs`, where the scope *and* the
window are automatic — but note a `filter=` there silently drops the scoping unless you include the session
id in the filter chain.) And **`createdAt` is not the call start** — verified live, it sat 42 s *after* the
first log line and the config load precedes it, so weight the window backwards as shown.

Two payoffs specific to a failing scenario-run: a `pipeline_error` decomposes into *"silent LLM response"*
→ *"**no fallback services available**"* → fatal — a **worker config gap** (no fallback model), not a
platform fault, and invisible in every other field. And
`level=debug&filter=|~:tools configured|No .* configured` prints the tool inventory at call setup, so a
prompt referencing an unwired tool is provable before the conversation even starts. Full technique, the
module map, and the `/logs/labels` trap: `commotion-debug/references/call-analyzer-api.md`.

This is what makes the handoff to `commotion-improve-worker` actionable: "failed the goal" sends it
guessing at the prompt; "called `get_policy_status`, got `not registered`, then asserted the answer
anyway" names the fix. The failure classes and the platform-artifact signature table are in
`commotion-debug/references/rca-taxonomy.md` — the same taxonomy applies to a scenario-run as to a
production call.

⚠ Some signals exist **only** in the transcript, never in `evaluationReasoning` — a mispronunciation, for
instance, shows up as the tester bot mis-hearing a term. That is precisely why this step exists.

If you defined metrics (Phase 1) and want their per-call scores (or to populate the Evals dashboard),
note metric evaluation is **async** (verified live): the sim's calls come back as eval-results in
`status: PENDING` with no scores until the evaluator runs. To read/force them: `commotion_request` `{
"method": "GET", "path": "/eval-result/session/<session-id>" }` where **`<session-id>` == the
scenario-run `id`** → if `body` is PENDING, `commotion_request` `{ "method": "POST", "path":
"/eval-result/trigger?voiceCallId=<voiceInteractionId>" }` (use the result's `voiceInteractionId`,
NOT `voiceCallMongoId`). `results[]` (`EvalMetricResultEntry`) gives `metricName`, `evaluation`,
`thresholdMet`. The pass-rate gate doesn't need this — it's for metric detail / the Evals dashboard.
Full plumbing in `references/simulation-and-results.md`.

## Phase 5 — Verdict + handoff

State the **pass-rate vs the user's target** (ask the target if they haven't said one — a common bar is
80, since `passRate` is a 0–100 percentage). If it clears the bar, you're done — the worker meets the quality goal at this version. If it's
below, summarize the failing scenarios and their reasons and hand off to `commotion-improve-worker`
(step 4), which diagnoses, edits a draft, and re-runs this skill until the bar is met. Keep the
`<sim-id>` and the failing-scenario analysis — they're the loop's state.

## Principles

- **The eval score is `SimulationResponse.passRate`** — report it with the raw count and the
  per-scenario reasons, not just a number.
- **Every channel simulates — always via `/simulation/run`.** Voice, chat and structured output all run
  here; the channel is the scenario's `aiAgentChannelType` (`CHAT` covers chat *and* SO). Never substitute
  `POST /aiworker/run` or a hand-made chat session: those are live runs (`callMode: COPILOT`), they appear
  under Observability's **live** filter, and they produce no pass-rate.
- **A generic simulation error is not a channel verdict** — read the run's own session
  (`/api/chat/session/<scenario-run-id>` → `errors[]`, or `/api/calls?requestId=`) before concluding.
  A worker LLM credential fault presents identically.
- Scenario-goal evaluation is automatic; eval-metrics are optional, secondary signal — don't block the
  run on them.
- **Sequential runs** — always check `GET /scenario-run/active` before starting; respect
  `maxScenarioRunLimit`.
- **Batch of 4** — never submit more than **4 total scenario-runs** in one `/simulation/run`; more
  overloads the websocket layer and runs fail with connection errors. For larger sets, run sequential
  batches of ≤4 and aggregate the pass-rates.
- Run against the **version under test** (the draft in the loop); everything is `(aiWorkerId, version)`
  scoped and `version` is on each record, not a list filter.
- Show every write (metric, run) before making it; surface backend errors and check the references.
