# The improvement loop — control, version-pinning, and the failure→fix taxonomy

How the iterate-until-threshold loop is controlled, and how to turn a failing scenario into the right
edit. The actual edit mechanics live in the `commotion-create-worker` references (cross-linked below);
this file is the **controller's** behaviour.

## The loop, precisely

```
baseline run (run-evals) ─► [ diagnose failures ─► edit DRAFT ─► re-run on DRAFT ─► compare ] ─► deploy?
                                       ^________________ while below threshold & rounds left ________|
```

State carried between rounds: `WORKER_ID`, the **draft version** under improvement, the latest
`SIM_ID`, the running **per-round summary** (round · edits · passRate · Δ), `threshold` (default **80**
— `passRate` is a **0–100 percentage**), `maxRounds` (default 3).

**Prerequisite (verified live):** the loop works on **any worker that has been deployed at least once** —
a draft version of an already-live worker is simulatable (a never-deployed worker returns *"Worker is not
available"*). **Channel is not a constraint:** voice, chat and structured-output all simulate via
`POST /simulation/run` (chat `passRate 100.0`, SO `PASS`, verified 2026-08-03).

## Stop conditions (check after every re-run)

1. `passRate ≥ threshold` → **success**, go to deploy gate.
2. `passRate < threshold` AND rounds remain AND `passRate` improved vs the previous round → **continue**
   (new failing scenarios drive the next round).
3. Otherwise (no rounds left, or no improvement, or a regression) → **stop and report**.

## The pass-rate must be measuring the worker (check before every round)

**`passRate`'s denominator is the *decided* runs, not `totalScenarios`** (verified live 2026-08-06: a
2-run batch with one `PASS` and one `Simulation Error` reported `passRate 100.0`, `passCount 1`,
`totalScenarios 2` — 1/1). So an unscored run does not drag the rate down; it **vanishes from the
denominator**, which is the more dangerous failure — a flattering rate can rest on a single decided
run. And when *nothing* is decided there is nothing to divide and the rate collapses to `0.0`,
indistinguishable from total failure (verified live 2026-07-28: **0 of 8 runs** decided).

Before treating a rate as a signal, count the runs behind it — `scenarioEvaluationResult` of
`PASS`/`FAIL`, not `ERROR` or `''`, and `scenarioRunStatusLabel` not `Evaluation Error` /
`Simulation Error`. Carry that count everywhere as "X% (n of N decided)".

Consequences for loop control:

- **Compare like with like.** A round-over-round delta is only meaningful if both rounds had a similar
  number of decided runs. Two rates built on different denominators are not comparable, and "improved vs
  the previous round" (rule 2 above) can be pure evaluator noise.
- **Zero decided runs is not a failure — it is a blocked loop.** Stop and report the harness problem.
  Iterating against an unmeasured score burns `maxRounds` and can leave the worker worse than it started.
- **A mute-caller round is also undecided** — `audioMetrics.userTurnCount: 0` with
  `stopReason: user_idle_timeout` means the personality wasn't `voiceEnabled`, so the scenario never ran.

## Regression guard

A round's edits must not lower the pass-rate. If a re-run comes back **worse** than the prior round:
- Treat the last edit set as a regression — **revert or reconsider it** rather than building on top of
  it (you're editing a draft, so reverting is just another `PUT`/agent edit, or re-create the draft
  from the last-good version).
- Don't count a worse round against `maxRounds` blindly — the point is to converge, not to burn rounds
  on changes that hurt. Re-diagnose and try a different fix.

Keep changes **small and attributable** each round: change the one or two things the diagnosis points
at, so a pass-rate move can be traced to a specific edit. A giant rewrite that moves the rate gives you
no signal about what worked.

## Draft-only discipline (the deploy rule)

- The live worker keeps serving its current version through the **entire** loop. All edits + all
  re-runs happen on a **draft** version.
- A live worker → `POST /aiworker/{id}/draft?version=<live>` mints a new draft version; iterate there.
- Only after the loop stops do you summarise and **ask** the user to deploy
  (`POST /aiworker/{id}/deploy?version=<draft>`). Never deploy inside the loop. This is the same
  deploy gate as `commotion-create-worker` Phase 10.
- Simulations can run against a draft version (the platform tests "whichever version is selected,
  including Draft") — so the draft-only loop is fully runnable.

## Version-pinning the test set

Scenarios and eval-metrics are `(aiWorkerId, version)`-scoped, and **list endpoints filter by
`aiWorkerId` only** (no `version` query param) — each record carries its own `version`.

- Run the simulation with `version` = the **draft** version under improvement.
- Ensure scenarios exist **for that version**. If a fresh draft version (N+1) doesn't carry the
  scenarios created at version N, regenerate/repoint the test set at N+1 via
  `commotion-generate-scenarios` before re-running.
- **Verified:** a simulation takes `scenarioId`s + a worker `version`, and a **draft version of an
  already-live worker is simulatable** — so run each round against the draft version you're editing.
  Eval-**metrics** are effectively worker-scoped (persist across deploys; created at the live version).
  Keep the *same* scenario set across rounds so pass-rate deltas are comparable; if a fresh draft
  version lacks the scenarios, recreate/point them at it (`GET /scenario?aiWorkerId=`, check `.version`).

## Failure → fix taxonomy (the full version)

Read each failing run's `failureReason` + `evaluationReasoning` (`GET /scenario-run?simulationId=`),
classify, and map to a fix. Reuse the create-worker references for the *how*. Some signals live in the
**transcript**, not the reasoning — a **mispronunciation** surfaces as the tester-bot's ASR mis-hearing a
term (the transcript wrote a different word than the worker said), so scan the conversation too, not only
`evaluationReasoning`.

| Symptom in `evaluationReasoning` | Root cause | Fix | Reference |
|---|---|---|---|
| Skipped a required step, wrong order, wrong tone, didn't follow the flow | Prompt gap | Tighten the agent **`instructions`** (add the step/rule; use the prompt-structure best practices) | agents-and-orchestration.md |
| Stated a backend fact (status/eligibility/price) it never fetched | Hallucination from un-wired data | Add the **grounding rule** (never assert un-fetched facts; say can't-verify) **and** wire the API as a tool | agents-and-orchestration.md + tools-and-capabilities.md |
| "Called API X" but nothing happened; agent fabricated `api_call`; looped re-asking | No real tool registered | **Register the API as a `custom-tool`** and reference `[tool:<action-name>]` in the prompt | tools-and-capabilities.md |
| Couldn't answer a question that's in the source docs | Missing/unbound grounding | **Attach + index knowledge**, bind it in the prompt with `[knowledge:<name>\|id:<id>]` | knowledge-and-rag.md |
| Switched language when the caller read digits/policy-no in English | Language rule missing | Add the **don't-switch-on-English-digits** rule to the prompt | aiworker-lifecycle.md |
| Re-asked for info already given; spun in a loop | Anti-repetition missing | Add the **don't-re-ask / call-each-tool-once / take-failure-path-once** rules | agents-and-orchestration.md |
| Re-asked for data that should be known up front (account/profile/tier), or forgot a value the caller gave earlier | No state variable | Define a **state variable** — `EXTRACTED` (capture-and-remember what the caller says) or `LOADED`/`PRE_LOADING` (fetch it once from a tool at call start) — and reference `[var:<title>]` in the prompt | settings-variables-pronunciation.md |
| Mispronounced a brand name, acronym, or SKU (TTS said it wrong) | No pronunciation entry | Add a **pronunciation dictionary** entry (usually `ALIAS`, e.g. `NPCL` → `"N-P-C-L"`) | settings-variables-pronunciation.md |
| Gave a generic or invented answer on a topic an attached **skill** already covers | The skill never loaded — its `description` didn't win the match | Rewrite the **`description`** as a routing signal (when to use it, in the caller's words), **not** the body. Confirm in Call Analyzer that no `read_skill` call happened that turn | skills-and-progressive-disclosure.md + debug/call-analyzer-api.md |
| Ignored rules that are present in the prompt, especially mid-prompt ones | Prompt too long to attend to | **Split self-contained topics into skills** (`POST /ai-worker-skill`, bind via `aiWorkerSkillIds`) so the prompt shrinks to routing and each body loads on demand | skills-and-progressive-disclosure.md |
| Answered a forbidden/off-limits topic, or gave advice it shouldn't | Guardrail gap | Add a **forbidden-word group** / a **custom guardrail** ("no financial/medical advice") | control-and-reliability.md |
| Blocked a legitimate request | Guardrail too strict | Turn off the offending **toxicity category** / narrow the forbidden-word group / fix the custom check | control-and-reliability.md |
| Fell for a jailbreak / prompt-injection / social-engineering attempt | Injection defense missing | Enable **`manipulationDetectionEnabled`** (advanced safety) | control-and-reliability.md |
| Drifted off-scope over a long conversation | No anti-drift layer | Enable **`focusGuardrailEnabled`** (advanced safety) | control-and-reliability.md |
| Ended the call prematurely or never ended it | Termination logic | Fix the end-call conditions in the prompt; ensure the `end_call` built-in is available | tools-and-capabilities.md |
| Slow turns flagged by a Latency metric | Architecture / verbosity | Trim the prompt, reduce tool round-trips, or raise the Latency threshold for a tool-heavy worker | run-evals/references/eval-metrics.md |

**A guardrail interception currently errors on the run/eval path (known code-side bug).** Verified live:
when toxicity / a forbidden-word group / a custom check / manipulation detection intercepts, the turn
should deliver your fallback text — but today it returns `status:"FAILED"` with a generic *"An error has
occurred … reference number …"* message (a backend issue, not the intended behaviour). So before
"fixing" a FAILED turn, check the input: on an abusive/off-limits/injection message that FAILED may just
be the guardrail intercepting. Only loosen the guardrail if it intercepted a *legitimate* request.

For a multi-agent worker, also check **routing**: if the wrong specialist handled a request, fix the
orchestrator (`workerLevelPrompt`) routing rules rather than the specialist's prompt.

## What this skill does NOT do

- **No server-side auto-tune.** There is no "improve prompt" endpoint in this backend — the diagnosis
  and the edit are *yours*. (Cekura's MCP exposes prompt-improvement endpoints; the Commotion dev3
  backend does not.)
- **No deploying inside the loop**, and **no editing a live worker directly** — always via a draft.
