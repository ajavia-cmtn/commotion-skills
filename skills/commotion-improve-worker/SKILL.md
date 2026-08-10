---
name: commotion-improve-worker
description: >-
  Iteratively improve a Commotion worker until it clears an eval-score threshold — read the failing
  scenarios from a simulation run, diagnose why each failed, edit the worker on a DRAFT version
  (prompt, tools, knowledge, guardrails), re-run the evals against that draft, and repeat until the
  pass-rate hits the target or a max-round cap, then deploy the improved version on approval. Use this
  whenever the user wants to improve / iterate / fix / tune a worker to raise its eval score, or "make
  it pass" / "get the pass-rate up" — e.g. "my renewal bot only passes 60%, improve it until it's 85".
  This is step 4 of the quality loop (create-worker → generate-scenarios → run-evals →
  **improve-worker**) and owns the loop. Calls the Commotion backend through the thin Commotion MCP server
  (OAuth — no API key in the transcript).
allowed-tools: Read, AskUserQuestion, mcp__commotion-dev3__commotion_request, mcp__commotion-tcuat__commotion_request, mcp__commotion-tcprod__commotion_request, mcp__commotion-dev3__commotion_schema, mcp__commotion-tcuat__commotion_schema, mcp__commotion-tcprod__commotion_schema, mcp__commotion-dev3__commotion_analyzer, mcp__commotion-tcuat__commotion_analyzer, mcp__commotion-tcprod__commotion_analyzer
---

# Commotion: Improve a Worker (the quality loop)

Close the loop: take a worker that doesn't pass its scenarios well enough, **diagnose** why from the
per-scenario failure reasons, **edit** it on a draft, **re-run** the evals, and repeat until the
**pass-rate** clears the target. You supply the judgment — reading the evaluator's reasoning, deciding
whether a failure is a prompt gap, a missing tool, missing grounding, or an over-strict guardrail, and
making the fix. There is **no server-side "improve prompt" button** — the improvement is your reasoning
plus the same editing machinery `commotion-create-worker` uses, run through the connected **Commotion
MCP** server's two tools (`commotion_request` / `commotion_schema`).

**Auth is automatic (browser login).** The Commotion MCP handles auth via OAuth: on first use it
opens a Commotion login in the browser, then attaches the user's token to every `commotion_request` /
`commotion_schema` call for you — the token never enters the conversation. Never ask the user for an
email/password or a token, and don't pass a `token` argument. If the two tools aren't available at
all, the MCP isn't connected — ask the user to add/authorize it via `/mcp`.

This is **step 4 of the worker quality loop, and it owns the loop**:

```
create-worker → generate-scenarios → run-evals → [improve-worker]
                                         ↑___________ repeat until pass-rate ≥ threshold ___________|
```

**Two hard rules (locked):**
1. **Draft-only loop — never auto-deploy mid-loop.** All iteration happens on a *draft* version: edit
   the draft → run evals against the draft → repeat. The live worker keeps serving its current version
   untouched until the very end.
2. **The gate is the scenario pass-rate** (`SimulationResponse.passRate`, a **0–100 percentage**). The
   loop stops when `passRate ≥ threshold` or a max-round cap is hit. The final improved draft deploys
   **only on the user's explicit "yes"** (same gate as create-worker Phase 10).

**Prerequisite (verified live):** evals need a **deployed** worker — the loop runs on a *draft version of
an already-live worker* (that draft is simulatable; a never-deployed worker returns *"Worker is not
available"*). **Channel is not a constraint:** voice, chat and structured-output workers all simulate via
`POST /simulation/run` (verified live 2026-08-03 — chat `passRate 100.0`, SO `PASS`), so **never convert a
chat worker to voice just to run the loop**. If the target has never been deployed, deploy it once first
(see `commotion-run-evals` prerequisites).

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

The user wants to raise a worker's eval score / make it pass / iterate on it. You need a baseline
simulation to diagnose from — if there isn't one, run `commotion-run-evals` first (this skill will
prompt you to). Scenarios must exist (`commotion-generate-scenarios`). The actual editing of the worker
(prompt, tools, knowledge, guardrails, deploy) reuses `commotion-create-worker`'s phases and references
— this skill is the **controller** around them. **For the whole build → test → improve pipeline in one
request, use the `commotion-quality-loop` orchestrator** (it invokes this skill as its improve step).

## How this skill talks to the platform (read first)

**Same transport** as the other skills — all platform I/O goes through the connected **Commotion MCP**
server (two tools, no scripts, no keys), one unified backend:

- **`commotion_request`** — one authenticated call: `{ "method": "GET|POST|PUT|DELETE", "path": "/…",
  "body": <JSON, for writes> }` → `{ "status", "body" }` (a non-2xx is **returned, not thrown** — read
  it and adjust). Pass a **path** (the base URL is fixed server-side) and the request payload as `body`
  — no temp files.
- **`commotion_schema`** — a bundled request schema: `{ "schema_name": "AiWorkerRequest" }` → the JSON
  Schema with its `$defs`. Any component name in the live spec works. **Never invent a field that isn't
  in the schema.**
- **`commotion_analyzer`** — one **GET** against the **Call Analyzer** plane: `{ "path": "/api/…" }` →
  the same `{ "status", "body" }` shape. Your diagnosis quality is capped by your evidence, and
  `evaluationReasoning` is the evaluator's *opinion* of a call; this is the call. Several rows of the
  taxonomy below are only **provable** here. ⚠ **Optional** — registered only where the Call Analyzer key
  is configured; if absent, say so once and diagnose from `evaluationReasoning` alone.

**Auth is automatic — there is no key.** The MCP client owns OAuth: the first time the
Commotion MCP is used it opens a Commotion login in the browser, then attaches the user's token to
every call. Never ask the user for a token. If the two tools aren't available, the Commotion MCP isn't
connected — ask the user to add/authorize it (in Claude Code: `/mcp` → **commotion** → Authenticate),
then continue.

**Read ids/fields from results, not `jq`:** `commotion_request` returns the parsed `body` — read the
value you need straight off it (e.g. a worker's live `version` is `body.version`) and feed it into the
next call's `path`. No shell, no `jq`.

References this skill leans on:
- This skill's loop control: `references/improvement-loop.md` (threshold, rounds, regression guard,
  version-pinning of the test set, the failure→fix taxonomy).
- The eval endpoint map: `commotion-generate-scenarios/references/eval-domain-api.md`.
- The actual editing machinery (reused, not duplicated):
  `commotion-create-worker/references/aiworker-lifecycle.md` (revert-to-draft, versions, voice),
  `…/agents-and-orchestration.md` (edit/POST agents, prompt rules),
  `…/control-and-reliability.md` (guardrails, fallback), `…/tools-and-capabilities.md` (wire tools),
  `…/knowledge-and-rag.md` (attach knowledge).
- Reading scores + failures: `commotion-run-evals/references/simulation-and-results.md`.
- The Call Analyzer plane (what each `?fields=` section returns, sizes, gotchas):
  `commotion-debug/references/call-analyzer-api.md`; the failure classes and the
  platform-artifact signature table: `commotion-debug/references/rca-taxonomy.md`. Both apply to a
  scenario-run exactly as to a production call — a simulation *is* a real call with a robot caller.

**Execution rules:** one phase at a time; **never deploy inside the loop**; show every edit before
making it; one round = diagnose → edit draft → re-run → compare.

## Phase 0 — Inputs and the stop condition  ·  HUMAN INPUT

Gather:
- **`aiWorkerId`** — the worker to improve (used as `<worker-id>` in later calls).
- **Baseline simulation** — the `<sim-id>` of the latest run (its failing scenarios drive round 1). If
  there isn't one, run `commotion-run-evals` now to produce one, then continue.
- **Threshold** — the pass-rate to reach (`AskUserQuestion`; **default 80** — `passRate` is a **0–100
  percentage**, not a 0–1 fraction).
- **Max rounds** — iteration cap so the loop terminates (`AskUserQuestion`; **default 3**).

Confirm the loop will run **draft-only** and only deploy at the end on their approval.

## Phase 1 — Diagnose the failures (each round)

Read the current run's failing scenarios: `commotion_request` `{ "method": "GET", "path":
"/scenario-run?simulationId=<sim-id>" }` → for each failing run in the result `body`, read
`failureReason` + `evaluationReasoning`.

> ⚠ **First separate "failed" from "never evaluated" — most of the loop's wasted rounds start here**
> (verified live 2026-07-28). A run whose `scenarioEvaluationResult` is **`ERROR`** or **`''`**, or whose
> `scenarioRunStatusLabel` is `Evaluation Error` / `Simulation Error`, carries **no verdict** — the
> evaluator crashed, often on a call that ran fine. In one session **0 of 8 runs** returned a usable
> verdict, and the resulting `passRate 0.0` looked exactly like total failure. Likewise a run with
> `audioMetrics.userTurnCount: 0` / `stopReason: user_idle_timeout` had a **mute simulated caller** (a
> personality with `voiceEnabled: false`) and never exercised the worker at all.
>
> **Never diagnose or edit against either.** There is nothing in the worker to fix, so a round spent on
> one is a round spent moving a prompt at random. Exclude them from the pass-rate, say how many were
> decided, and if *none* were, stop and report the harness problem instead of iterating — a threshold
> loop cannot converge on a score that isn't measuring the worker. `commotion-run-evals` Phase 4 has the
> bucketing table; to recover real verdicts from the transcripts, use `commotion-debug` (it reads Call
> Analyzer directly).

Classify each failure and map it to a fix (the **failure → fix taxonomy** — full version in
`references/improvement-loop.md`):

| Failure pattern (from `evaluationReasoning`) | What **proves** it (Call Analyzer) | Fix | Where |
|---|---|---|---|
| Agent missed a step / wrong flow / wrong tone | the `transcription[]` turns themselves | Edit the agent **`instructions`** | agents-and-orchestration.md |
| Asserted a backend fact it never fetched (hallucination) | `toolCallMetrics[]` **empty**, or its `result` is an error, while the transcript states the fact as certain | Add/strengthen the **grounding rule**; wire the API as a **tool** | agents + tools-and-capabilities.md |
| "Called an API" but nothing happened / looped | `toolCallMetrics[].result == "Error: function '<name>' is not registered."`, or `[tool:<name>]` in the prompt with no match in `callMetadata.registered_tools` | **Register the API as a tool**, reference `[tool:<action>]` | tools-and-capabilities.md |
| Couldn't answer from source material | no knowledge grounding in `context[]`; nothing retrieved | **Attach + index knowledge**, bind it in the prompt | knowledge-and-rag.md |
| Flipped language on English-spoken digits | the switch is visible turn-by-turn; `sessionState.language` | Add the **don't-switch-on-digits** prompt rule | aiworker-lifecycle.md |
| Over-blocked a legitimate request / under-blocked a bad one | the interception in the transcript (⚠ a guardrail block can surface as a generic `FAILED`) | Tune **guardrails** (toxicity category toggles, forbidden-word groups, custom checks, advanced safety) | control-and-reliability.md |
| Re-asked for info already given / looped | repeated near-identical `assistant` turns — **but check `contextCorruption.detected` first** | Add the **anti-repetition / call-once** prompt rules | agents-and-orchestration.md |
| Mispronounced a term | **only** in the transcript — the tester bot mis-heard the word the worker said | **Pronunciation dictionary** (`ALIAS`) | settings-variables-pronunciation.md |
| "Too slow" | `latencyMetrics.summary` vs `toolCallMetrics[].latency_seconds` — a slow *tool* and a slow *model* need opposite fixes | depends: tool/upstream, or model + prompt length | tools-and-capabilities.md |
| Talked over the caller / long silences | `turnTimeline[]` overlap, `audioMetrics.silenceRatioPercent` / `botRatioPercent` | barge-in / VAD + speaking-plan config, not prompt prose | control-and-reliability.md |
| Call terminated mid-conversation (`pipeline_error`, unexplained `stopReason`) | **the error logs** — `/logs?session_id=<scenario-run-id>&level=error`. Verified live, this decomposes into `[ALERT] Silent LLM response` → **`No fallback services available`** → fatal: a **missing fallback model**, which nothing else reveals | Configure a **fallback model** | control-and-reliability.md |
| **Not the worker at all** | `contextCorruption.detected`, `llmFallbackEvents` with `fallbackSucceeded: true`, `state_load_errors`, `audioLossPercent`, mute caller | **nothing** — report it | `commotion-debug/references/rca-taxonomy.md` §2 |

To fill that middle column, pull each failing run's call — the scenario-run `id` **is** the call's
`requestId`:

```
commotion_analyzer { "path": "/api/calls?requestId=<scenario-run-id>&limit=1" }         → the callId
commotion_analyzer { "path": "/api/call/<call-id>?fields=transcript,metrics,analysis" }  → ~12 KB
```

(Add `?fields=config` separately — it is ~41 KB on its own — when you need `context[]` or
`registered_tools`. Never request all five sections at once; that gets truncated.)

**Classify before you edit.** The last row is the one that costs rounds: a platform artifact
prompt-fixed is a wasted round *and* a false fix. The full signature table is
`commotion-debug/references/rca-taxonomy.md` §2 — the same taxonomy applies to a scenario-run as to a
production call.

Prioritize the fixes that clear the most failing scenarios. Present the diagnosis to the user, quoting
the evidence — the turn, the tool `result`, the metric value — not a paraphrase.

## Phase 2 — Edit on a DRAFT (never on live)

Agents/config are editable only on a **draft**. Establish the draft version you'll edit and hold it as
`<draft-version>` (Phase 3 and Phase 5 reuse it):

- **Worker is live** → read its current version, then mint a new draft version (e.g. N+1) alongside the
  still-serving live version:
  1. `commotion_request` `{ "method": "GET", "path": "/aiworker/<worker-id>" }` → the live version is
     `body.version` (call it `<live-version>`).
  2. `commotion_request` `{ "method": "POST", "path": "/aiworker/<worker-id>/draft?version=<live-version>" }`
     → the new draft's version is `body.version` (call it `<draft-version>`).
- **Worker is already a draft** → there's no live version to read (`GET /aiworker/{id}` is **live-only**
  and 400s on a draft-only worker). Get the draft's version from the version history — `commotion_request`
  `{ "method": "GET", "path": "/aiworker/<worker-id>/versions" }` → in the result `body`, find the entry
  under the **`items`** wrapper whose `status` is `DRAFT` and read its `version` as `<draft-version>`;
  edit in place. (Only one draft can exist at a time, so there's at most one DRAFT entry. Don't POST
  `/draft` again. A superseded live version shows status **PAUSED**.)

Then apply the diagnosed fixes on that draft version.

Apply the diagnosed fixes using the **create-worker machinery** (don't reinvent it):
- **Prompt** → **`PUT /aiagent/{id}`** with the revised `instructions` (full body — resend name,
  description, `agentType`, model config, triggers, or they reset). The prompt renders/edits in the UI
  and the **`aiAgentId` is unchanged, so your scenarios keep working** — verified live 2026-08-07. This
  replaces the old delete-and-re-POST rule; see agents-and-orchestration.md.
- **Tools** → create on the draft (`POST /ai-worker-tool/...`) and reference by action name in the
  prompt — see tools-and-capabilities.md.
- **Knowledge** → attach + index, bind in the prompt — see knowledge-and-rag.md.
- **Guardrails / fallback** → `PUT /aiworker/{id}` (full PUT, resend kept fields + `version`) — see
  control-and-reliability.md.
- **Language / voice config** (e.g. add a language) → `PUT /aiworker/{id}` full body with
  `workerVoiceSettingsRequest.workerVoiceConfiguration.allowedLanguages` updated (e.g. `["en","hi"]`) +
  a language-mirroring rule in the agent prompt. **Resend the fields you want to keep** (name, setup,
  guardrails) or they reset. Verified live: adding `hi` + the prompt rule made the agent switch to
  Hindi mid-call and stay in Hindi on English-spelled emails.
- **Agent type change** (e.g. CHAT_AGENT→VOICE_AGENT) → **delete the agent and re-POST** it with the
  new `agentType`. A `PUT` that changes the type is rejected: *"Cannot change agent type…"*.

> ⚠ **After any delete + re-POST, re-point your scenarios or Phase 3 cannot even start** (verified live
> 2026-07-28). Since prompt edits now go through `PUT`, this should only bite you on an **agent-type
> change** or a deliberate agent replacement — but when it does bite, it is silent. The re-POST assigns a
> **new `aiAgentId`**, and every scenario still stores the old one, so
> `POST /simulation/run` returns `500 Simulation trigger failed. Please try again.` — indistinguishable
> from the known transient flake, and retrying never clears it. Fix: read the new id from
> `GET /aiagent?workerId=<worker-id>&version=<draft-version>`, compare it to each scenario's `aiAgentId`
> (`GET /scenario/<id>`), and `PUT /scenario/<id>` with the new value (full replace — resend the fields
> you keep). Note `PUT /scenario` **silently ignores a changed `version`**; `aiAgentId` is what actually
> binds a scenario to the version under test.
>
> Re-confirmed live while re-POSTing: `POST /aiagent` **requires `agentType`** (400 *"Agent creation
> failed: agentType is required."*), and you read it back as **`aiAgentType`** — the `agentType` key
> itself is `null` in every response. `DELETE /aiagent/{id}` needs `?version=`. Both are already spelled
> out in `commotion-create-worker/references/agents-and-orchestration.md`; the new part is only that a
> re-POST inside *this* loop invalidates the scenarios. **The cheapest way to avoid all of this is to
> not re-POST** — `PUT` handles prompt edits and keeps the id.

**Show every edit before making it.** Make the smallest set of changes that addresses the round's
failures (so you can attribute the pass-rate change to them — see the regression guard).

## Phase 3 — Re-run the evals against the draft

Point the test set at the draft version and re-run, reusing `commotion-run-evals`:

- Ensure scenarios exist **for the draft version** (a draft-of-a-live worker is simulatable; a sim
  takes scenarioIds + the worker `version`). If the fresh draft version lacks them, recreate/point the
  test set at it via `commotion-generate-scenarios` — see `references/improvement-loop.md`.
- Check `GET /scenario-run/active?aiWorkerId=<worker-id>` (sequential), then `POST /simulation/run` with
  `version` = the **draft** version. **Cap each sim at ≤4 total scenario-runs** — more overloads the
  websocket layer and runs fail with connection errors; for a larger set, run sequential batches of ≤4
  and aggregate (see `commotion-run-evals`). Poll `GET /simulation/{id}` to COMPLETED and read the new
  `passRate`. This becomes the round's result and the next round's `<sim-id>`.
  **A ≤4-run voice batch takes ~2 min** — first poll at ~45 s, then every ~30 s, and quote the duration
  from **`timeToComplete` (ms)**, never your own elapsed time. `duration: 0.0` on a still-`RUNNING` run
  is normal, not a stall. `passRate`'s denominator is **decided** runs, so report it as "X% (n of N
  decided)". Never abandon a round and tell the user to run it in the UI.

## Phase 4 — Loop control (the decision each round)

- **`passRate ≥ threshold`** → **stop: success.** Go to Phase 5.
- **Below threshold, rounds remaining, and `passRate` went up** (regression guard: an edit that *lowered*
  the rate is reverted/reconsidered, not kept) → **back to Phase 1** with the new failing scenarios.
- **Below threshold and (no rounds left or no improvement)** → **stop: report.** Summarize where it
  plateaued and the remaining failures; suggest the next manual step (e.g. a tool that needs a real
  endpoint spec, or knowledge that doesn't exist yet).

Keep a **per-round summary table**: round · what changed · `passRate` (and Δ) · scenarios fixed/broken.
Show it to the user as the loop progresses.

## Phase 5 — Deploy the improved draft  ·  ALWAYS ASK FIRST

Only after the loop stops:
- Summarize the final draft (version, the edits made across rounds, final `passRate` vs target).
- `AskUserQuestion`: deploy now or keep as draft.
- On a clear **yes**: `commotion_request` `{ "method": "POST", "path":
  "/aiworker/<worker-id>/deploy?version=<draft-version>" }`.
- Otherwise leave it as a draft (it persists; the user can deploy later). Confirm live with
  `commotion_request` `{ "method": "GET", "path": "/aiworker/<worker-id>" }`.

## Principles

- **Draft-only, deploy-at-end.** Never deploy inside the loop; the live worker is untouched until the
  user approves the final version. This mirrors create-worker's deploy gate.
- **Gate on the pass-rate**; diagnose from `failureReason` + `evaluationReasoning`. Fewer, targeted
  edits per round so you can attribute the change.
- **Regression guard + max rounds** keep the loop honest and terminating — don't keep an edit that
  lowered the score; always have a round cap.
- **Reuse, don't reinvent** — the editing is create-worker's machinery; this skill is the controller.
- Version-pin the test set to the draft under test; show every edit; deploy only on explicit "yes".
