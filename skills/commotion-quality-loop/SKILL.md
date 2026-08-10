---
name: commotion-quality-loop
description: >-
  Run the FULL Commotion worker quality loop end-to-end — build (if needed) → generate test scenarios →
  run evals → improve — iterating until the scenario pass-rate clears a threshold, then deploy on
  approval. This is the single entry point that orchestrates the four specialist skills in order and
  owns the "keep improving until it passes" control flow. Use this when the user wants the WHOLE
  pipeline in one go — e.g. "build a voice bot that books test drives and make it pass 90%", "test and
  improve my renewal worker until the pass-rate is 80%", "set up X and evaluate it end to end". **Also
  the default entry point for a production / production-ready / production-grade worker, or any long,
  detailed problem statement describing a real system to ship** — a production use case implies build +
  test + iterate, not just build, so prefer this over commotion-create-worker whenever the goal is a
  worker meant to actually go live for real traffic (even if the user doesn't name a pass-rate). For a
  SINGLE step, defer to the specialist instead: build only → commotion-create-worker; scenarios only →
  commotion-generate-scenarios; run evals only → commotion-run-evals; improve only →
  commotion-improve-worker. Calls the Commotion backend through the thin Commotion MCP server (OAuth — no
  API key in the transcript).
allowed-tools: Read, AskUserQuestion, Skill, mcp__commotion-dev3__commotion_request, mcp__commotion-tcuat__commotion_request, mcp__commotion-tcprod__commotion_request
---

# Commotion: Quality Loop (end-to-end orchestrator)

Drive a worker from an idea to a **measurably good** deployment in one flow. This skill doesn't
re-implement anything — it **sequences the four specialist skills** and owns the top-level loop:

```
create-worker → generate-scenarios → run-evals ──► improve-worker ──► deploy (on approval)
   (if needed)                          ▲                    │
                                        └──── repeat until passRate ≥ threshold or max rounds ────┘
```

You invoke each specialist via the **Skill** tool, carry the shared state between them (worker id,
version, scenario ids, latest simulation id, pass-rate), and make the loop/threshold decisions here.
Every write stays human-approved, and **deploy is always user-gated** — same discipline as the
specialists.

**Auth is automatic (browser login).** The Commotion MCP handles auth via OAuth — on first use it
opens a Commotion login in the browser and attaches the user's token to every call, for this skill
and every specialist it invokes via **Skill**. Never ask the user for an email/password or a token,
and don't pass a `token` argument. If the tools aren't available, the MCP isn't connected — ask the
user to authorize it via `/mcp`.

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

## When to use this (vs a specialist)

Use the loop when the request spans the whole pipeline ("build **and** test **and** improve until it
passes"). If the user only wants one step, invoke that specialist directly and stop. Don't run the
whole loop for a single-step request.

## Hard prerequisites (verified live — the loop can't run without these)

Automated evals need a **deployed** worker. There is exactly **one** hard prerequisite:
- The worker must have been **deployed (live) at least once** — a never-deployed worker returns *"Worker is
  not available"* and can't be simulated. (After that, a **draft version** of it *can* be simulated,
  which is what lets the improve loop run on a draft.)

**Channel is NOT a prerequisite.** Voice, chat and structured-output workers all simulate through
`POST /simulation/run`; the channel comes from each scenario's `aiAgentChannelType` (`VOICE`, or **`CHAT`
for chat *and* structured output**). Verified live 2026-08-03: a chat worker scored `passRate 100.0` and
an SO worker returned `PASS`. **Never voice-enable a chat worker just to run the loop** — build the test
set on its own channel.

If the worker hasn't been deployed, the loop's job is to get it deployed before generating scenarios.

## Transport — same as the specialists

This skill mostly delegates to the specialists via the **Skill** tool; its own platform I/O goes
through the connected **Commotion MCP** server's two tools (`commotion_request` / `commotion_schema`)
— no scripts, no keys. **Auth is automatic:** the MCP client owns OAuth (it opens a Commotion login
in the browser once), so there's no key to provide. If the tools aren't available, the Commotion MCP
isn't connected — ask the user to add/authorize it (in Claude Code: `/mcp` → **commotion** →
Authenticate), then continue.

Confirm the Commotion MCP is connected with one read before you start: `commotion_request`
`{ "method": "GET", "path": "/aimodel" }` (a 2xx means you're good). The specialists share the same
connection, so this check covers the whole loop.

## Phase 0 — Scope the run  ·  HUMAN INPUT (batched, minimal)

Establish, asking only for what you can't infer:
- **The goal** (what the worker should do) and whether a worker **already exists** (`aiWorkerId`) or
  must be built.
- **Threshold** — the scenario pass-rate to reach (default **80**; `passRate` is a **0–100
  percentage**).
- **Max rounds** — improvement-iteration cap (default **3**).

Confirm the plan in one line ("build a voice worker for X → generate scenarios → evaluate → improve on
a draft until pass-rate ≥ 80 (max 3 rounds) → deploy on your yes") before running.

## Phase 1 — Ensure a deployed VOICE worker (entry point)

- **No worker yet** → invoke the **`commotion-create-worker`** skill to build one. Steer it to a
  **voice** worker (evals need voice), and see it through to a **deploy** (the loop needs a live
  version). Capture the `aiWorkerId`.
- **Worker exists but is chat / never deployed** → get it to a deployed voice worker: enable voice on
  a draft and deploy (create-worker covers the edit→deploy flow). 
- **Deployed voice worker exists** → capture its `aiWorkerId` + the version you'll test (default the
  live version). Proceed.

Do not continue to Phase 2 until the prerequisite (deployed voice worker) holds.

## Phase 2 — Build the test set

Invoke the **`commotion-generate-scenarios`** skill for this worker + version. It creates the
simulated-caller **personalities** (voice-enabled) and the **scenarios** (cover happy + failure +
jailbreak/edge paths). When it returns, capture the **scenario ids** and the version they were
created at.

## Phase 3 — Baseline eval

Invoke the **`commotion-run-evals`** skill to run the scenarios as a simulation and report the
**baseline pass-rate** (optionally defining eval-metrics for richer signal). Capture the `SIM_ID`,
the `passRate`, and the per-scenario failures (the diagnosis fuel). Report the baseline vs the
threshold.

> ## ⛔ Gate the whole loop on the baseline being *measured* (verified live)
>
> This skill's entire control flow is a comparison against `passRate`, so it fails hardest when that
> number isn't measuring the worker. Two ways it isn't, both seen live on dev3 2026-07-28:
>
> - **The evaluator produced no verdict.** `scenarioEvaluationResult` comes back `ERROR` or `''` with
>   `scenarioRunStatusLabel` of `Evaluation Error` / `Simulation Error`. Observed: **0 of 8 runs decided**,
>   yielding `passRate 0.0` on calls that had run 40–60s perfectly well.
> - **The simulated caller was mute.** `audioMetrics.userTurnCount: 0` and
>   `stopReason: user_idle_timeout`, because the personality had `voiceEnabled: false`.
>
> So before Phase 4, require from `commotion-run-evals` **how many runs were decided**, not just the rate.
> If **none** were, **do not enter the improve loop** — it would iterate against a score that cannot move,
> burning every round and possibly leaving the worker worse. Report the harness problem and stop: fix the
> personality (`commotion-generate-scenarios` Phase 2) or escalate the evaluator bug. If only *some* were
> decided, say so in the baseline and treat round-over-round deltas with matching caution — two rates on
> different denominators are not comparable.
>
> **The specialists read the calls, not just the scores.** Both `commotion-run-evals` and
> `commotion-improve-worker` now pull each failing scenario-run's actual call from **Call Analyzer** (a
> simulation is just a real call with a robot caller), so the failures they hand back should already name
> the evidence — the tool that errored, the fabricated turn, the mute caller — rather than only the
> evaluator's opinion. If a round comes back with bare "failed the goal" reasons, ask for that evidence
> before spending the next round on a guess. When the evaluator is the thing that's broken,
> `commotion-debug` is the specialist that works purely from the calls. It is not a phase of this loop,
> but it is the tool that can tell you whether a `0.0` means
> "broken worker" or "broken evaluator".

## Phase 4 — Improve loop (the core)

- If the baseline was **not measured** (no decided runs) → **stop**, per the gate above. Do not iterate.
- If **baseline `passRate` ≥ threshold** → skip to Phase 5 (already meets the bar).
- Else → invoke the **`commotion-improve-worker`** skill, passing the worker id, the baseline
  `SIM_ID`, the **threshold**, and **max rounds**. It owns the round-by-round mechanic — diagnose the
  failing scenarios → edit the worker on a **draft** → re-run evals against that draft → repeat, with a
  regression guard — until `passRate ≥ threshold` or max rounds. **The whole loop stays on a draft; it
  never auto-deploys.** When it returns, capture the final draft version + final pass-rate + the
  per-round summary.

Surface the per-round summary table (round · edits · pass-rate Δ) to the user as it progresses.

## Phase 5 — Deploy the result  ·  ALWAYS ASK FIRST

Summarise the outcome (final version, pass-rate vs threshold, what changed across rounds). Then
`AskUserQuestion` to deploy the improved version live (Deploy now / Keep as draft). Only on a clear
yes, deploy it (the improve-worker / create-worker deploy step). If the loop hit max rounds without
clearing the bar, say so plainly and hand back the remaining failures + a recommended next step
(e.g. a tool that needs a real endpoint, or knowledge that doesn't exist yet) — don't deploy a
regression.

## Phase 6 — Confirm + report

Confirm the live worker (`GET /aiworker/{id}`) and give a final report: the worker, the final
pass-rate, the deployed version, and (if metrics were defined) how to read the Evals dashboard
(metric evaluation is async — see `commotion-run-evals`).

## Principles

- **Orchestrate, don't duplicate.** Each phase is "invoke specialist X, capture its output, decide."
  The specialists carry the field shapes and gotchas; this skill carries the *sequence* and the
  *threshold loop*.
- **Prerequisite first:** a deployed **voice** worker, or the evals can't run.
- **Threshold is a 0–100 pass-rate**; the loop is **draft-only**; **deploy is always user-gated**.
- **Carry state** between skills (worker id, version, scenario ids, SIM_ID, pass-rate, threshold, max
  rounds) so each specialist gets what it needs without re-asking.
- If any specialist reports a blocker it can't resolve (e.g. the worker can't be made voice, or a fix
  needs data that doesn't exist), stop and report — don't loop pointlessly.
