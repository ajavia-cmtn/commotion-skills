# Changelog

## 2026-08-16 — 2.0.2 — Call Analyzer is now per-user RBAC (token-based), not blanket-admin

The Call Analyzer plane moved from a server-side global api key (blanket admin, cross-workspace) to
per-user, token-based RBAC enforced in Call Analyzer. Skills docs updated to match: the endpoint-map
contract ([`commotion-debug/references/call-analyzer-api.md`](skills/commotion-debug/references/call-analyzer-api.md))
replaces its "No token / admin-scoped" claims with token-based, per-user, **role-scoped** semantics
(admin sees all; analyst/reader get reduced/redacted/workspace-scoped data), and documents a **`403`
= no Call Analyzer access** response. `commotion-debug` gains a permission-denied (403) stop branch
distinct from tool-absence; the loop skills (`run-evals`, `improve-worker`, `generate-scenarios`,
`create-worker`) now treat a `403` or role-reduced view the same as tool-absence — say so once and
fall back to `evaluationReasoning`, never block. Docs only; no tool contract change.

## 2026-08-12 — 2.0.1 — Interruption config: two mutually exclusive modes, and the orphan-block trap

Triaging a bug ticket ("with connector we are able to configure Interruption Word Settings and
Interruption Prediction together when it's restricted to do so") turned up a gap in these skills: the
**speaking plan was undocumented entirely**, so nothing told an agent that interruption has two modes
and that only one may be enabled. Verified live against dev3 **and** tcuat on 2026-08-12:

- **The backend does enforce the mutex.** Both flags true → `400` on `POST /aiworker` *and*
  `PUT /aiworker/{id}`, with the same message the UI toasts: *"Interruption Word Settings and
  Interruption Prediction can't be enabled at the same time."* The check is unconditional — it fires
  even behind `interruptionConfig.enabled: false`, so the combination can't be staged. `PUT` is a true
  replace (no merge escape), and `AiAgentRequest` carries no interruption config, so worker
  create/update is the only write path. The ticket as written does not reproduce.
- **What does get through is an orphaned word-settings block.** Validation guards only the two
  booleans, not the config they gate: `interruptionWordSettingsEnabled: false` while still sending
  `numberOfWords` + `interruptionIgnoreTerms`, alongside `interruptionPrediction.enabled: true`,
  returns **200 and persists both blocks**. The worker then opens with the keyword list *and* the
  probability threshold populated, and the UI refuses to save until a human turns one off — a worker
  the connector wrote but can no longer edit there. That is the reported symptom.

What changed in the skills:

- **`aiworker-lifecycle.md` gained "Interruption config — two mutually exclusive modes"** — the
  `speakingPlanRequest.interruptionConfig` path, a mode/flag/fields table, the 400, and the rule that
  matters: **switching modes means clearing the losing mode's fields, not just its flag.** Verified
  GOOD payloads for both modes (each round-tripped live; an empty `interruptionIgnoreTerms` normalizes
  to `null` and `interruptionPrediction: {enabled: false}` normalizes to `null`, so neither leaves
  residue), plus which mode to prefer and how `duckingEnabled` composes with prediction.
- **`commotion-improve-worker`'s failure→fix taxonomy** row for *"talked over the caller / long
  silences"* now points at that section instead of `control-and-reliability.md`, which has no
  barge-in/VAD content at all — the loop was being sent to the wrong file for exactly the fix that
  produces this state.

**Not fixed here, and it can't be:** this stops *these skills* from writing the state; it does not
close the hole. Any other client of the API can still persist it, and existing workers already in that
state are untouched. The durable fix is server-side normalization — when
`interruptionWordSettingsEnabled` is false, clear `numberOfWords`/`interruptionIgnoreTerms` on write,
or reject the payload when they're non-empty and prediction is enabled. Two questions remain open for
the BE/FE owners: whether the voice pipeline honours the `false` flag when ignore terms are present
(if not, both features are live on real calls — the exact restricted combination), and whether the UI
derives each section's toggle from data presence rather than the boolean.

## 2026-08-10 — 2.0.0 — Skills, the first message, and ACW: three backend features the skills could not reach

The backend shipped three capabilities the plugin had **no** coverage of. A grep proved the gap:
zero hits for `greetingInstructions` or any agent-skills feature, and exactly two passing mentions of
"ACW" that referenced it without ever saying what it was or how to wire it. So a worker built by
`commotion-create-worker` could not use any of them, and `commotion-improve-worker` had no lever to
reach for when a prompt had simply grown too long to attend to.

All three were established by **building a real worker end to end on dev3 on 2026-08-10** — `Skills
Probe Clinic` plus its `[ACW] Skills Probe Clinic` target — rather than by reading schemas. That
mattered: three of the rules below are things the schema states and the server does not enforce, and
one is a field the schema advertises on an object that rejects it.

- **Skills are progressive disclosure, not prompt injection — and that is the whole point.** New
  reference `skills-and-progressive-disclosure.md`. A skill (`POST /ai-worker-skill`, id returned as
  `aiWorkerSkillId`) is a named, described instruction block. At runtime the system prompt carries only
  an `<available_skills>` manifest of names and descriptions; the platform auto-registers a built-in
  **`read_skill(name)`** tool and the body arrives only when a turn calls it. Measured on the probe
  worker: **870 input tokens on the manifest turn, 1053 after the body loaded**, with the agent quoting
  figures that exist nowhere but inside the skill. This is the answer to "the prompt is too long".

- **The `description` is the routing signal, and it decides whether a skill works at all.** It is the
  only thing the model sees when choosing to load. Documented as a tool-description discipline (say
  *when* to use it, in the caller's words) with the corollary now in the improve-worker failure→fix
  taxonomy: when an agent invents an answer a skill already covers, **fix the description, not the
  body** — the body was never read.

- **`aiWorkerSkillIds` binds; `[skill:<name>]` only hints.** Unlike every other mention token, the
  skill token is never expanded — it stays literal in the resolved prompt. The manifest is built from
  the id array on `AiAgentRequest`. Both are now in the token table in `tools-and-capabilities.md`,
  with the asymmetry called out, because a `[skill:X]` with no bound id is a dangling reference that
  produces no error at all.

- **Two silent-acceptance traps, both verified.** A skill id belonging to a *different* worker is
  accepted into `aiWorkerSkillIds` with a `200` and never resolves. And `greetingInstructions`, whose
  schema says only state and system variables may be referenced, happily stored a greeting containing
  `[skill:…]` and `[knowledge:…|id:…]` — the caller would simply hear the raw token. Both are now
  authoring rules the skills police, plus a new readiness-gate check in Phase 9 that catches the first
  one, since nothing else will.

- **ACW belongs on the worker, not the agent — the schema misleads here.** `AiAgentRequest`
  advertises `advancedSettingsRequest.acwMetadataRequest`, but a voice agent rejects the entire block:
  `400 "Advanced settings is not supported for VOICE_AGENT type."` Since ACW is a voice feature, the
  real path is `workerVoiceSettingsRequest.workerAdvancedVoiceSettingsRequest.acwMetadataRequest`. New
  Phase 8.7 covers the fixed build order — create the `[ACW] <name>` worker, configure its agent,
  **deploy it**, then reference it — because a draft target fails with `400 "ACW live worker not found
  with ID: …"`. Also documented: sending `voiceAgentType` alongside the ACW block drags in telephony
  validation (`connectionId cannot be blank`), so omit it.

- **Skill ids survive a version bump**, same as agent ids: `POST /aiworker/{id}/draft` carried both
  skills into v1 with `aiWorkerSkillId` unchanged, so `aiWorkerSkillIds` keeps resolving across
  revert → edit → redeploy. Folded into the existing id-stability paragraph rather than restated.

- **Skills get a dangling-reference check of their own**, the analogue of `registered_tools` for tools:
  `/api/chat/session/<id>` → `config.skills[]` carries `isReferenced` and `referencedBy[]`, and a
  `read_skill` call is visible in `runs[].tools`. An answer on a skill's topic with no `read_skill` in
  that turn is an ungrounded answer. Added to `commotion-debug/references/call-analyzer-api.md`.

### Round 2 — how the reference tokens actually resolve

Prompted by an observation that skills render as chips in the editor while tools and knowledge look
like plain text, a second pass added a tool and a knowledge source to the same probe worker and traced
each. The answer is that the chip/text distinction is cosmetic and the real mechanism is uniform.

- **No mention token is string-substituted; every one becomes a tool call.** `[skill:…]` →
  `read_skill`, `[knowledge:…|id:…]` → `search_knowledge_base`, `[tool:…]` → the registered action.
  The model always sees the raw token in the prompt — the UI chip is a display layer over the same
  literal text, which is why `finalInstructions` still shows the token.
- **The `|id:` in a knowledge token is the retrieval filter, which finally explains the token-shape
  asymmetry the reference documented but never justified.** The observed call is
  `search_knowledge_base(query, filters:[{"key":"id","value":"<id from the token>"}])`; the id scopes
  the search to one document. A wrong id doesn't error — the search returns empty and the agent
  answers from nothing.
- **Tokens work from inside a skill body, all three kinds.** A `[tool:…]` that appeared *only* in a
  skill body was still registered and called; so was a `[knowledge:…|id:…]`. This extends the
  compose-ability claim from skills-calling-skills to the full token set.
- **Attaching a skill is now documented as two mandatory steps**, not one: bind the id in
  `aiWorkerSkillIds` *and* reference it from the agent's `instructions` with `[skill:<name>]`. The id
  array is what makes the skill reachable; the token is what puts it in the agent's own procedure at
  the right point in the flow, keeps the routing legible to whoever reads the prompt next, and makes
  it reviewable in the Phase 9 gate. The readiness check now runs in both directions.
- **⚠ Tool action names are re-minted per worker version**, and the platform rewrites `[tool:…]`
  tokens (in agent prompts *and* skill bodies) when a draft is forked: the same tool was
  `check-slot-availability-2135` at v1 and `-2136` at v2. Live references keep working; hand-copied
  ones do not.
- **⚠ A rule inside a skill body is not a stronger rule.** A skill saying "Never invent slots" did not
  stop the agent inventing three after the tool returned none. The grounding rule belongs in the agent
  prompt, where it covers every turn.
- **⚠ In Call Analyzer, a custom tool is listed under an opaque MCP-server hash**, not its action name
  — the names are inside `functions_vs_metadata`. Searching `config.tools[]` for the action name finds
  nothing and reads as "not wired".

### Two corrections picked up along the way

- **`POST /aiagent` on a `SINGLE_AGENT` worker is refused**, not merely redundant:
  `400 "Cannot create another agent. Single Agent setup allows only one agent."` The reference said the
  default agent was the convenient path; it is the only one.
- **`languageModelSettingsRequest.temperature` is required** on an agent write —
  `400 "…temperature is required."` — despite not appearing in the schema's `required` list.

## 2026-08-07 — 1.7.0 — One plugin, three environments: dev3, tcuat and tcprod connectors

The skills were pinned to dev3 by exactly one line — `plugin.json`'s MCP server URL. Everything else
was already environment-agnostic: a sweep for absolute URLs across `skills/` returns only `react.dev`
and `example.com` placeholders, because every Commotion call goes through `commotion_request` /
`commotion_schema` / `commotion_analyzer` and therefore inherits whatever server it is routed to.

- **`plugin.json` now registers three MCP servers** — `commotion-dev3`, `commotion-tcuat`,
  `commotion-tcprod` — each with its environment's URL hardcoded. One repo, one branch, one install
  link, and **nothing for the user to configure**: install the plugin, then `/mcp` → authenticate the
  connector for the environment you work in. Connectors you never authenticate sit idle. An earlier
  draft used a `${COMMOTION_MCP_URL:-…}` environment variable; that was dropped because it made every
  non-dev3 user set a variable before launching, which is exactly the friction this avoids.

- **Tool names are now environment-scoped**, since MCP namespaces tools by server:
  `mcp__commotion-tcuat__commotion_request` and so on. Every skill's `allowed-tools` was expanded to
  cover all three connectors (e.g. create-worker went from 7 entries to 13).

- **Each skill now opens with "Pick the environment before the first tool call"** — the first `##`
  section in all six. Several identical tool sets is a real hazard when one of them is production, so
  the rule is explicit: the user's words win; exactly one connector available means use it; several
  and no instruction means ask once and then use that one connector for the whole task; state the
  environment in the first substantive reply and again before the first write to a production
  connector; and **never retry a failed call on a different connector** — a `401` means that connector
  is not authorized, not that another environment should be tried.

  The rule identifies the environment by **the environment word in the tool name, not a fixed
  prefix**, because the namespace differs by client: `mcp__commotion-tcuat__commotion_request` in
  Claude Code (from `plugin.json`), but `mcp__claude_ai_Commotion_Agent_UAT__commotion_request` in the
  Claude desktop app, where it comes from whatever the administrator named the connector. An earlier
  draft hardcoded the Claude Code prefixes and was simply wrong in the desktop app — it told Claude to
  look for names that do not exist there. Two consequences for admins: a desktop connector binds to
  the plugin **by URL**, so it must match `plugin.json` exactly; and its **name must make the
  environment obvious**, since that string is the only signal the skills get.

- **`commotion-debug` handles an absent Call Analyzer.** The analyzer plane is opt-in per environment,
  and its route is not yet exposed on tcuat/tcprod. The skill now stops and says Call Analyzer is
  unavailable on the connected environment, offering commotion-improve-worker instead, rather than
  reconstructing a call from BE REST data and presenting that as RCA.

- **De-dev3'd the prose that describes where the skills point** — all six `description:` frontmatters,
  `README.md` (new **Choosing an environment** section with the connector table), `marketplace.json`,
  and `references/api-and-auth.md`. Statements recording *what was observed on dev3* are deliberately
  untouched: "verified live against dev3 on <date>" is a historical fact, and the known-dev3-bug notes
  (e.g. `mcp-server` tool create returning 500) may not hold elsewhere.

Server-side counterpart: `commotion-mcp` 0.2.0 adds the tcuat/tcprod chart environments. The BE OAuth
provider is not yet released on either, so authenticating those two connectors will not complete yet;
`commotion-dev3` works today.

## 2026-08-07 — 1.6.0 — `PUT /aiagent/{id}` renders in the UI: the delete-and-re-POST rule for prompts is retired

The platform's prompt editor now reflects a `PUT`, which reverses the single most invasive rule in these
skills. Since 1.0 every skill said an agent's prompt only renders in the UI if the agent was **created**
via `POST /aiagent`, so a prompt edit meant *delete the agent and re-POST it* — which mints a **new
`aiAgentId`** and silently breaks everything pinned to the old one (scenarios being the one that bit us,
repeatedly). That cost is now unnecessary. Verified live against dev3 on 2026-08-07, both workers
confirmed in the UI by a human:

- **`PUT` into a blank editor populates it.** Chat worker `PUT UI Test 20260807` (`6a7566d1…28cb`): the
  auto-provisioned "Chat Agent" was never deleted or re-POSTed — a single `PUT /aiagent/{defaultId}`
  carrying `instructions` — and the prompt **rendered**. This is the exact case 1.x called impossible.
- **`PUT` also overwrites an existing prompt.** Voice worker `POST UI Test Voice 20260807`
  (`6a75680e…59fe`): default deleted → `POST /aiagent` with a prompt (rendered) → `PUT` with different
  text → the editor **showed the new text**. So it isn't a fill-once behaviour and the editor doesn't go
  stale. Both agents were then `PUT` a third time and tracked correctly.

What changed in the skills:

- **Prompt writes are `PUT /aiagent/{id}` everywhere** — `commotion-create-worker` Phase 6, plus the
  `[tool:…]` / `[knowledge:…]` / `[var:…]` binding recipes in `tools-and-capabilities.md`,
  `knowledge-and-rag.md` and `settings-variables-pronunciation.md`, which had all routed through the
  POST-create rule. `PUT` is a **full replace**: resend `name`, `description`, `agentType`, model config
  and triggers, not just `instructions`.
- **`SINGLE_AGENT` is now one call, not three.** `PUT` the auto-provisioned default (its `agentType` is
  already right for the channel) instead of GET → DELETE → POST. Same for the `STRUCTURED_OUTPUT`
  variant. `MULTI_AGENT` repurposes the default as specialist #1 and POSTs the rest — only `POST` can
  *add* agents.
- **Delete + re-POST is now scoped to what `PUT` genuinely can't do:** changing `agentType` (still
  immutable via `PUT` — `400 "Cannot change agent type…"`) or removing an agent. The `aiAgentId`-churn
  warnings in `commotion-improve-worker`, `commotion-debug` (§4a Trap 1) and `commotion-run-evals` are
  kept but re-scoped to those cases — and `repro-and-gates.md`'s stated tension between "keep the repro
  scenario byte-identical" and "re-point `aiAgentId`" **dissolves**, because the prompt fix no longer
  touches the id.
- **The FAQ exception is retired.** FAQ instructions are still `PUT`-only (`POST /aiagent` rejects FAQ
  types; `/aiagent/standard` has no `instructions` field) — that just isn't a visibility problem anymore.
  Flagged as *not separately re-verified on an FAQ agent*: same `PUT` path, expected to behave the same.
- The superseded 2026-07-21 findings are left in place in `settings-variables-pronunciation.md` and
  `tools-and-capabilities.md`, marked as superseded with the new result beside them — they were correct
  observations of a platform that has since changed, and the pairing is what makes the change legible.

### Round 2 — every agent type, and a full deploy cycle (same day)

A second live pass on workers `All Agent Types PUT Test 20260807` (voice, MULTI_AGENT) and
`SO Agent PUT Test 20260807`, each state confirmed in the UI by a human. Five agents — `VOICE_AGENT`,
`CHAT_AGENT`, `FAQ_CHAT`, `FAQ_VOICE`, `STRUCTURED_OUTPUT` — were written `HI` + a `[tool:…]` token,
`PUT` to `Hello` + a `[knowledge:…]` token, deployed, then `PUT` again to `hey aarya` and redeployed.
Every state rendered. Four further findings, all new:

- **`PUT` visibility is not type-specific, and FAQ is no longer an exception.** Both `FAQ_*` types —
  which can *only* be written by `PUT`, since `/aiagent/standard` has no `instructions` field — render
  and re-render. 1.6.0's hedge ("not separately re-verified on an FAQ agent") is now discharged
  directly, and the swap from a `[tool:…]` to a `[knowledge:…]` token showed the editor tracks a change
  of mention-token family, not just prose.
- **`FAQ` and `CUSTOM` are enum values with no create path.** Attempting all seven types on one worker:
  `POST /aiagent` → `400 "Only VOICE_AGENT, CHAT_AGENT & STRUCTURED_OUTPUT agent type is supported."`;
  `POST /aiagent/standard` → `400 "Only FAQ_CHAT & FAQ_VOICE agent type is supported."` Neither door
  opens. The skills previously listed `FAQ` as creatable via `/aiagent/standard` alongside
  `FAQ_CHAT`/`FAQ_VOICE` — **that was wrong** and is corrected in `agents-and-orchestration.md`
  (new per-type creatability table) and the create-worker FAQ section.
- **A `STRUCTURED_OUTPUT` agent needs its own worker.** It can't join a voice or multi-agent worker:
  `400 "Structured output is not enabled on the associated worker. Enable structured output on the
  worker first."`, and the flag itself is *"mutually exclusive with voice. Supported for Single Agent
  and Workflow setups."* Hence the companion worker in this test.
- **The deployed-worker edit cycle is id-stable, and old versions go `PAUSED`.** A live worker rejects
  the edit (`400 "Agent can only be updated/deleted when worker is in draft status."`), so the cycle is
  revert → `PUT` at the new version → redeploy. Reverting **preserved every `aiAgentId`** across all
  five agents on both workers (only `createdDate` restamped), and after deploying v1 the history read
  `[{v0, PAUSED}, {v1, LIVE}]` — a status the lifecycle reference didn't document (it said "superseded").
  Together with `PUT` as the prompt path, **no routine operation churns an agent id anymore** — which
  was the original worry that prompted this whole investigation.

## 2026-08-05 — 1.5.2 — The web crawler shipped: it is real, it lives on `km-setting`, and 1.5.1's "UI-only crawler" rule was wrong

The backend added a **web crawler API**, which invalidates the headline claim of 1.5.1 below. The old
golden rule 9 — *"NEVER use `WEBSITE_CRAWL` … there is no crawler behind it"* — was correct when written
but is now actively harmful: it would steer you into hand-uploading page snapshots when a real crawler
exists. Verified live against dev3 on 2026-08-05 with four purpose-built workers.

- **The crawler is configured on the `km-setting` plane, not the knowledge plane.** `webCrawlerConfig` was
  added to `KMSettingUpdateRequest`, so the whole feature is one call:
  `PUT /aiworker/km-setting/setting/{settingId}` with `{entityId, webCrawlerConfig: {...}}`. Fields:
  `enabled`, `seedUrls`, `crawlDepth` (1–5), `maxPages` (max 100/run), `sameDomainOnly`,
  `excludeUrlPatterns`, `syncFrequency` (`DAILY`/`WEEKLY`/`MONTHLY`), `authType`
  (`NONE`/`BASIC`/`COOKIE`), `authCredentials`, `lightweightMode`, `respectRobotsTxt`; plus read-back
  `lastCrawlInfo` (`lastCrawlStatus` `PENDING`→`CRAWLING`→`INDEXING`→`COMPLETED`/`PARTIAL`/`FAILED`, and
  `pagesDiscovered`/`Crawled`/`Indexed`/`Skipped`/`Failed`/`Removed`). Note the **path list is unchanged at
  220** — the crawler added no endpoint, only a schema block, which is why a path-only sweep misses it.
- **The crawler creates and indexes its own knowledge entries — you never touch the knowledge plane.** A
  completed run took **35s** (`COMPLETED`, `pagesCrawled: 1`, `pagesIndexed: 1`) and produced an item named
  from the page's real `<title>` (`Quick Start – React`), `sourceType: MARKDOWN`, `fileName`
  `Quick_Start___React.md`, a **system**-generated blob key (`demo/<orgId>/system/<rand>-https___react.dev_learn.md`),
  already at `Completed`. So no `file-upload`, no `/knowledge/bulk`, no `/knowledge/index` for a crawl.
- **Hand-creating a `WEBSITE_CRAWL` item still fails, and that is the part 1.5.1 got right.**
  `POST /knowledge/bulk` with a URL in `sourceUrlIdentifier` is accepted `200`, lands `Draft`, flips to
  `Failure` — the URL is percent-encoded onto the blob container (golden rule 5). Reproduced on two workers;
  enabling `webCrawlerConfig` first changes nothing. The two planes are unrelated.
- **There is no manual crawl trigger over the API.** The UI has a "Trigger Web Crawler Sync" button with no
  API counterpart: no crawl route among the 220 paths, `/v3/api-docs/swagger-config` exposes only the
  `public` group, and six plausible routes all return a generic Spring `404`. **Over the API a crawl only
  lands on its schedule (`DAILY` ≈ midnight)** — the skills now require saying so, and offering the
  `TEXT_UPLOAD` snapshot recipe when the user needs content immediately.
- **⚠ New platform bug found: a partial `chunkingConfig` on the `km-setting` PUT permanently destroys the
  worker's KB index.** `chunkingConfig` has six file-type blocks; send it with any missing and the absent
  ones become `null`, a new `indexName` is minted, index creation fails
  (`"Please modify settings for index creation"`), and `indexId`/`collectionName` are wiped. **Unrecoverable**
  — re-PUTting all six still fails. Isolated by elimination across three workers: `entityType`,
  `collectionName`, `indexingConfig`, `embeddingConfig` and repeat PUTs are each individually harmless.
  Its nastiest consequence is silent: on a worker in this state the **crawl still runs and fetches**, but
  its item sticks at `Draft` and is never indexed. The skills now mandate the minimal PUT (only the fields
  being changed) and explicitly warn against the "GET it and send it all back" pattern, which is what
  triggers it.
- **`commotion_schema` can serve a stale spec.** `KMSettingUpdateRequest` came back *without*
  `webCrawlerConfig` from cache and only revealed it with `{"refresh": true}` — worth knowing before
  concluding a field doesn't exist.
- **Two smaller findings.** `pagesDiscovered` returns `null` on a `COMPLETED` run while every other counter
  populates; and `lightweightMode: true` skips JS rendering, so a `crawlDepth: 1, maxPages: 5` crawl of a
  SPA (`react.dev/learn`) discovers no links and yields exactly one page.
- **The skills now require telling the user what was arranged.** Crawling is the one knowledge source where
  a successful setup looks identical to a silent failure: you write a config, get a `200`, and the KB stays
  empty. So `SKILL.md` (Phase 7 + Principles) and a new **"REQUIRED — report the arrangement back to the
  user"** block in `knowledge-and-rag.md` mandate saying that crawling is configured in the worker's
  settings, naming the **actual** frequency (daily/weekly/monthly), and stating that entries are created and
  indexed automatically — with an empty knowledge list until the first run being expected, not a failure.
- **Pre-deploy readiness corrected for crawler-only workers.** The Phase 9 gate demanded every knowledge
  item be indexed, which a crawler-configured worker cannot satisfy — it legitimately has **zero** items
  until its first crawl. An empty list is no longer a readiness failure there; instead verify `km-setting`
  `status: SUCCESS` + `webCrawlerConfig.enabled`, and warn that the worker has no grounding until the crawl
  runs.
- Rewritten in `commotion-create-worker`: golden rules 2/9 and the `aiWorkerKnowledgeType` enum note, a new
  **"Web crawler"** section, the `km-setting` section (crawler + the `chunkingConfig` warning), the
  endpoints table, the reframed "Fetched web page" recipe (now a snapshot, not a crawler substitute), plus
  Phase 7's source list and two Principles in `SKILL.md`. Also fixed the golden rules' numbering, which ran
  6, 9, 10, 7, 8.

**Still unverified:** that the `DAILY` schedule fires *on its own*. Both crawls observed on 2026-08-05 were
manual UI syncs (the 07:16 UTC run reconciles exactly with the UI's "Job completed at 12:46 PM" IST). A
clean API-only worker — `6a72e5030e114ddf3fc81fa9`, never opened in the UI, `lastCrawlInfo: null`, zero
knowledge items — is parked on dev3 awaiting its first scheduled run to confirm.

## 2026-08-03 — 1.5.1 — QA bug sweep: chat/SO simulations, the UI-only crawler, Commotion-only providers, the voice catalogue, and the agent's missing model

Five QA-reported bugs, all of them **skill defects rather than platform defects** — the API supported the
right thing in four of the five cases and the skills either forbade it, missed the endpoint, or wrote a
silently-broken payload. Everything below was verified live against dev3 on 2026-08-03 with purpose-built
workers, and each fix names the evidence.

- **The "simulations are voice-only" rule was false, and it was the root of bug 1.** Six files asserted, as
  a verified hard constraint, that only voice workers can be simulated. They cannot all have been right:
  `POST /simulation/run` accepted a **chat** worker and returned **`passRate 100.0`**, and a
  **`STRUCTURED_OUTPUT`** worker returned **`scenarioEvaluationResult: "PASS"`** — both echoing
  **`onlyChatScenarios: true`**. There is **one** simulation endpoint for every channel; the channel comes
  from each **scenario's `aiAgentChannelType`**, and `GET /scenario/dropdown-config` has always listed
  `channelType: [VOICE, CHAT]`. Corrected in `commotion-run-evals` (SKILL + `simulation-and-results.md`),
  `commotion-generate-scenarios` (SKILL + both references), `commotion-improve-worker` (SKILL +
  `improvement-loop.md`), `commotion-quality-loop`, `commotion-debug`, and `commotion-create-worker`.
- **Why the false rule survived: a generic error was mis-generalised.** A failing chat run reports only
  *"An error has occurred during simulation. Please contact support with reference number …"* with
  `duration 0.0`. On the worker that produced it the simulation had in fact **started correctly** and
  injected the scenario's `userScript` verbatim; the real cause sat in the session's own `errors[]` —
  `litellm.AuthenticationError: Incorrect API key provided: sk-… model: openai/qwen3-next`, a **worker LLM
  credential fault**. The skills now require reading `/api/chat/session/<scenario-run-id>` (or
  `/api/calls?requestId=`) before drawing any conclusion from a generic simulation error, and say
  explicitly that such an error is never a statement about the channel.
- **`STRUCTURED_OUTPUT` lives on the CHAT channel.** Per the schema, `aiAgentChannelType` *"must be CHAT for
  CHAT_AGENT / STRUCTURED_OUTPUT and VOICE for VOICE_AGENT"*, and a freshly provisioned SO agent reports
  `aiAgentChannelType: "Chat"`. There is no SO channel value. Added as a table to
  `commotion-generate-scenarios`, which now also requires setting `aiAgentChannelType` explicitly on every
  scenario and reading `channelTypeLabel` back (it is the **Type** column in the Scenarios UI).
- **Never run a live session in place of a simulation.** The observability half of bug 1: because chat sims
  were believed impossible, chat workers were "tested" via `POST /aiworker/run` and hand-invented session
  ids (`fix-orch-1`, `diag-orch-1`, … all still visible on the reported worker). Call Analyzer tags those
  `callMode: COPILOT`, so they land under Observability's **live** filter with no `scenarioEvaluationResult`
  and no pass-rate. Now called out as a prohibition in `commotion-run-evals`, `commotion-generate-scenarios`
  and `commotion-debug`.
- **Personality `voiceEnabled` is channel-specific.** The rule "personalities must be `voiceEnabled: true`"
  is a **voice** rule; for chat/SO, `false` is correct and sufficient — both passing runs above used a
  `voiceEnabled: false` persona. Previously stated unconditionally, which pushed toward voice-converting
  chat workers.
- **`WEBSITE_CRAWL` is unusable over the API — bug 2.** *(Partly superseded by 1.5.2: the backend has since
  added a real crawler on the `km-setting` plane. The "no crawler exists / UI-only" conclusion below is no
  longer true; the "never hand-create a `WEBSITE_CRAWL` item" prohibition still is.)* Searching the whole
  spec (220 paths) there is **no
  crawl endpoint and no crawl-config field anywhere**; `WEBSITE_CRAWL` exists only as an enum value. The
  crawler is a UI-only service. Confirmed by a controlled A/B on one worker against the same page
  (`https://react.dev/learn`): `WEBSITE_CRAWL` + a raw URL → **`Failure`**; fetch-then-`TEXT_UPLOAD` →
  **`Completed`**. The mechanism is now documented — the backend concatenates the storage container with
  `sourceUrlIdentifier` verbatim, so a URL there becomes
  `…/digitalassets-storage-data/https%3A%2F%2Freact.dev%2Flearn`, a blob that never existed. The item is
  accepted with `200`, sits at `Draft`, then flips to `Failure` on index. `knowledge-and-rag.md` gains the
  prohibition, the A/B table, a **"Fetched web page"** recipe, and the rule that **`sourceUrlIdentifier` is
  always a storage key, never a URL** — plus a note to delete any `Failure` crawl item rather than leave it
  in the KB, and that `pageNumber` is **0-based** (`pageNumber=1` returns `[]` and reads as "no knowledge").
- **Always a Commotion provider for LLM / STT / TTS — bug 3, and it is worse than reported.** The report was
  that credentials go unselected while the model gets backfilled. The cause: **the primary LLM, TTS and STT
  request objects have no credential field at all** (only *fallback* models and the simulator/eval-metric
  `LLMConfig` do), and **no endpoint lists or creates** client provider credentials —
  `AiModelResponse.credentialId` is documented *"null for platform models"*. So a non-Commotion provider can
  only ever be written credential-less. And the backend does not object: `PUT /aiworker` with
  `provider: "eleven-labs"` for **both** TTS and STT returned **`200`, no error, no warning**, with no
  credential anywhere in the response — rendering as *Provider: filled · Credential: blank · Model: filled*,
  saved and unrunnable. New **"Providers and credentials"** section in `aiworker-lifecycle.md` with a
  field-by-field credential table, a matching box in `control-and-reliability.md`, and top-level rules in
  `commotion-create-worker`: use `commotion` / `commotion-tts` / `commotion-llm`, never write a
  provider/model you cannot credential, and tell the user an external provider is UI-only rather than
  half-configuring it.
- **The voice catalogue exists: `GET /aiworkervoice` — bug 4.** Voices were assumed UI-only because the
  endpoint is one word with no separator, so searching for `/voice` finds nothing — and
  `aiworker-lifecycle.md` actively misdirected to `GET /aimodel`, which returns *language* models and has
  no voices in it. `/aiworkervoice` filters by `providerId`/`modelId`/`languageId`/`accentId` and returns
  `voiceId`, `voiceName`, a prose description, and `languageIdToVoiceUrlMap` — per-language **sample audio**
  to play to the user. All five `commotion-laya-v1-5` voices are now tabled (Komal, Raj, Poornima, Tanya,
  **Abhay**), with the note that the long-quoted "verified-good" UUID `d6d81480…` is simply **Poornima**.
  Also corrected: `accentIdToAccentDropdownOutputMap` is **language accents**, not voices, and will never
  yield a `voiceId`. The skills now query the catalogue and offer named choices when asked to pick a voice.
- **Every agent needs `modelConfigurationRequestList` — bug 5.** An agent created without it lands with
  `modelConfigurationResponseList: []` — genuinely no model, a blank *Provider / Credential / Model* panel,
  and an agent the user cannot run. The primary model is a **top-level array** on `AiAgentRequest`
  (`{id, modelCode, providerCode}`, all three required, `id` from `/aimodel`);
  `LanguageModelSettingsRequest` has **no** primary-model property, so nesting it there — the natural guess,
  since it mirrors the *worker* shape, which genuinely is nested — is **silently dropped**: `200`, sibling
  settings round-trip, only the empty response array reveals the loss.
- **Two corrections to the agent/worker model story.** Measured in both directions, so neither is inference:
  (a) **agents inherit nothing from the worker's model choice** — a chat worker set to `commotion-3.6-27b`
  and a voice worker set to `commotion-3.6-27b` both produced default agents on **`commotion-3.6-35b`**, the
  platform default; the previous claim that an agent "still inherits the worker-level default at runtime"
  was wrong, and the panel being blank is a real broken state, not a display gap. (b) A **`VOICE_AGENT` does
  carry** a top-level `modelConfigurationRequestList` (its auto-provisioned one came back populated), so the
  earlier "assume the voice agent follows the worker LLM" speculation is replaced with the measurement. This
  makes the mandatory-model rule apply to **chat, voice and SO alike** — and it matters most on the
  **delete-default-then-POST** path the skills require for UI-visible prompts, which is exactly where the
  model goes missing.
- **Smaller verified fixes.** `POST /aiagent` **requires `maximumOutputTokens`** (omitting it returns
  `400 "languageModelSettingsRequest.maximumOutputTokens is required"`). The `/aiagent` list filter is
  **`workerId`**, not `aiWorkerId` — the wrong name is silently ignored and returns a global, paginated
  list. And a new standing principle across `commotion-create-worker`: **read the response back after every
  write** rather than trusting the `200`, because several fields here are silently dropped when mis-shaped
  or accepted-but-broken.

## 2026-07-27 — 1.5.0 — a debug skill: RCA a real call, reproduce it, then fix the worker

All five existing skills point forward — build, generate scenarios, run evals, improve. Nothing pointed
backward at a worker already serving traffic: there was no path from "this call misbehaved" to "the worker
is fixed", and the skills carried no observability endpoints at all. Call Analyzer's GET APIs are now
exposed through Kong and carry exactly the signals an RCA needs, so this release adds the missing
direction — plus the MCP tool to reach it.

- **New skill `commotion-debug`** — take one misbehaving **live** interaction and end with a worker that
  no longer misbehaves. Seven phases: scope the defect → pull the evidence from Call Analyzer → root
  cause + failure class (user confirms) → **reproduce as a failing simulation** → fix on a DRAFT →
  verify → regression sweep + report, deploying only on an explicit yes. References:
  `call-analyzer-api.md` (endpoint map, `fields=` sections, measured payload sizes, gotchas) +
  `rca-taxonomy.md` (7 failure classes + a 10-row platform-artifact signature table) +
  `repro-and-gates.md` (repro construction, the stochastic gates, the overfitting check, loop bounds).
  Structured after Cekura's `cekura-fixing-prod-issues`, with the stochastic gates its successor added.
- **It is deliberately NOT part of the quality loop.** The loop is a build-time pipeline over a *test
  set*; this starts from *production traffic*. So the skill carries neither the "step N of the quality
  loop" framing nor the "for the whole pipeline use the orchestrator" line every specialist has, and
  `commotion-quality-loop` is untouched — it still sequences exactly four specialists. The two connect
  only through an explicit escalation: when the defect turns out to be breadth rather than one bug (no
  single root cause, a plateau across 3 rounds, or a broadly failing regression sweep), `commotion-debug`
  hands `commotion-improve-worker` the worker id and its latest `<sim-id>` as a baseline and says why.
- **Reproduce before you fix — the gates are rates, not verdicts.** A voice simulation runs real audio, so
  one run false-passes often enough to be misinformation. **Reproduced = the repro scenario FAILS in ≥ 2
  of 4 runs** and the failure is visible turn-by-turn in a failing transcript; **fixed = it PASSES in
  ≥ 3 of 4**. Always reported as the rate (`3/4 pass`). The asymmetry is deliberate: reproducing wants
  sensitivity, verifying wants confidence. N is 4 because that is the per-simulation websocket cap from
  1.4.1 — and the `COMPLETED`-instantly / `passRate 0.0` / `avgLatencyInMillis null` signature is called
  out as *the runs did not happen*, explicitly **not** a reproduction.
- **New MCP tool `commotion_analyzer`** (needs commotion-mcp ≥ the 2026-07-27 Call Analyzer plane) — a
  **GET-only** reader for Call Analyzer, added to the skill's `allowed-tools`. It could not ride
  `commotion_request`: the plane's Kong route authenticates with a static `apikey` header and **rejects
  the caller's bearer** (verified live — `apikey` alone `200`; `apikey` + `Authorization: Bearer …`
  `401`), its payloads are browser-sized, and it is read-only at the gateway. The tool has no `method`
  argument, so nothing in this skill can change production through it, and it returns the same
  `{status, body}` envelope as `commotion_request` plus `truncated`/`dropped` when a response was
  shortened. **A truncated body is incomplete evidence** — the skill re-queries narrower rather than
  concluding from the fragment.
- **Measured payload sizes drive the fetch strategy** (28-second call, verified live): the five-section
  fetch is ~60 KB and **gets truncated**, of which `config` alone is ~41 KB. So Phase 1 fetches
  `transcript,metrics,analysis` (~12 KB) first and pulls `config` or `timeline` separately, only when the
  symptom implicates them. `/logs` is **1.6–3.9 MB** unshaped (Loki's row limit is hardcoded 5000
  server-side and not overridable from the query), so it always needs a narrow window *and* a line filter.
- **Gotchas the reference now carries, all verified live.** `?fields=all` **does not work** — the code
  does exact set-membership on the comma-split value, so it silently returns the shell; name each
  section. A call's `workerId` is **`<aiWorkerId>_<version>`** — strip the `_N` for the BE-plane id, and
  query *both* forms when counting a worker's calls, because both occur. `timeWindow` accepts only `5m`,
  `15m`, `1h` and **silently ignores** anything else. `/logs` requires `start`+`end` and treats
  `session_id` as a line-*contains* filter, not a label match. `/api/filter-options` returns empty lists
  by design.
- **The repro bridge is flagged as unconfirmed.** `POST /scenario/generate-from-conversation` needs a BE
  `conversationId`, not the Call Analyzer `callId`; the mapping goes through
  `GET /conversation/worker-conversations?workerId=` matched on `voiceInteractionId`. That single link is
  the one step not yet verified live, so Phase 3 ships with an explicit fallback (author the scenario from
  the verbatim transcript via `commotion-generate-scenarios`) rather than a guess.
- **Exercised end to end against dev3 (2026-07-28) on a deliberately broken worker**, and the findings
  are folded back into all three references plus `commotion-run-evals` and `commotion-improve-worker`.
  A voice worker was built with an agent prompt referencing `[tool:get_policy_status]` (never registered)
  **plus** instructions to state a renewal amount, a debit, an expiry date and an emailed certificate as
  confirmed fact and never to admit uncertainty. 12 scenario-runs over 3 simulations produced 10 real
  calls. The defect reproduced exactly — `toolCallMetrics[0].result == "Error: function
  'get_policy_status' is not registered."` followed immediately by *"I can confirm that your renewal
  amount of four thousand nine hundred and ninety nine rupees has already been debited"* — and after the
  fix on draft v1 all four verify runs passed with *"I cannot verify your renewal details on this call"*
  and zero fabrications. **Repro 1/1 failed → verify 4/4 passed.** Four corrections came out of it:
  - **⚠ The evaluator returned no usable verdict on 0 of 8 runs**, so the gates cannot read
    `scenarioEvaluationResult`. It is not limited to PASS/FAIL — it also comes back **`ERROR`** or an
    **empty string**, with `scenarioRunStatusLabel` of `Evaluation Error` / `Simulation Error`, on calls
    that ran 40–60s with complete transcripts (one cause is an evaluator-side bug: *"Goal completion
    evaluation failed … `channel_types.0` Input should be 'voice' or 'chat' [input_value='customer']"*).
    Both gates now **bucket runs, exclude no-verdict ones from the denominator, and derive the missing
    verdicts from the Call Analyzer transcript** via `/api/calls?requestId=<scenario-run-id>`, reporting
    `n of 4 decided` and saying when a verdict was transcript-derived. *"The evaluator did not run"* and
    *"not reproducible"* are now explicitly different findings.
  - **⚠ The 1.4.1 failure signature is ambiguous and was overstated.** `COMPLETED` + `passRate 0.0` +
    `avgLatencyInMillis null` does **not** prove the runs didn't happen — observed live on a simulation
    whose four calls all existed with 42–57s durations. Discriminate on per-run `duration` and on whether
    the calls exist in Call Analyzer. Corrected here and in
    `commotion-run-evals/references/simulation-and-results.md`.
  - **⚠ Delete + re-POST of an agent silently invalidates every scenario pinned to it.** The re-POST
    assigns a **new `aiAgentId`** while scenarios keep the old one, so `POST /simulation/run` returns
    `500 Simulation trigger failed. Please try again.` — indistinguishable from the known transient flake,
    and retrying never clears it. Both `commotion-debug` and `commotion-improve-worker` now require
    re-pointing the scenario (`PUT /scenario/<id>` with the new `aiAgentId`) after a prompt edit. Also
    verified: `PUT /scenario` **silently ignores a changed `version`** — `aiAgentId` is what binds a
    scenario to the version under test.
  - **⚠ A mute tester bot invalidates a whole simulation.** One 4-run sim produced four 54–56s calls in
    which the caller never spoke (`userTurnCount: 0`, `userAudioSeconds: 0.0`,
    `stopReason: user_idle_timeout`) because every available personality had **`voiceEnabled: false`**.
    Phase 3 now pre-flights `GET /personality` for `voiceEnabled: true`, and the taxonomy carries the
    signature as a harness artifact — a voice gate cannot be read from a mute caller.
  - New platform-artifact rows also captured: `stopReason: pipeline_error` on four otherwise-clean
    completed conversations; tool-call markup spoken aloud (`Have a wonderful day! [tool:end_call{reason:`);
    and `llmFallbackEvents` carrying *"LLM produced empty response (no content, no tool calls)"* with
    `fallbackSucceeded: false`. Plus: a dangling `[tool:…]` reference is provable **statically** from the
    prompt against `callMetadata.registered_tools`, and `registered_tools` always contains the built-in
    `end_call` — so `initialContextCost.toolCount: 1` means "one built-in", not "a custom tool is wired".
- **Logs are promoted to a first-class signal, with a technique** (exercised live on the broken worker's
  call). The first cut treated `/logs` mainly as a size hazard to be capped; in practice it is the **best
  evidence for a config, startup or run-time fault**, because it is the only surface that says *why a call
  ended*. Measured on one 40-second call: `session_id` alone returns **763 rows / 489 KB**, while adding
  **`level=error` returns 3 rows / 2.3 KB** — ~200× smaller, and it contains the root cause. So the
  guidance is now *narrow first*: `level=error`, then widen with `filter=|~:ERROR|ALERT|fatal` (the warning
  trail) or `q=Transcript` (the conversation as the pipeline logged it). `commotion-debug` Phase 1 reads
  error logs on **every** call rather than only on an audio/crash symptom, and `commotion-run-evals` /
  `commotion-improve-worker` gained the same query for any `stopReason` they can't explain.
  - **A `pipeline_error` is usually a worker config gap, and only the logs show it.** The chain reads
    `[ALERT] Silent LLM response from CommotionLLMService#0` → **`[ALERT] No fallback services
    available`** → `ErrorFrame(error: LLM produced empty response (no content, no tool calls),
    fatal: True)` → `PIPELINE ERROR … terminating call`. The model returned empty — recoverable — and
    **no fallback model was configured to catch it**, so it killed the call. The taxonomy row for
    `pipeline_error` previously said "don't treat as a behavioural defect"; it now sends you to the logs
    and names the fix (configure a fallback). The `llmFallbackEvents` row likewise splits on
    `fallbackSucceeded`: `true` is a platform artifact, **`false` is a config gap**.
  - **`sessionId` is the one scoping key, and it works for voice and chat alike** — reframed from the
    earlier voice-centric *"`session_id` is the call's `requestId`"*. For voice the call's `requestId`
    **is** its session id (the logs say so: `workspace.save: Saved session <requestId>`,
    `session_tracker: Session unregistered: ws_<requestId>`); for chat it is the `sessionId` in the path.
    The endpoints differ only in ergonomics, now documented as a table: voice **requires** an explicit
    `session_id` plus `start`+`end` (`400` otherwise), while `/api/chat/session/<id>/logs` **auto-scopes
    from the path and derives its own window**, so the whole escalation ladder runs with no parameters at
    all. Chat logs come from the same stream (`service_name=voice-ai`) and the same config-logging modules,
    so every technique — including the config mining — applies to chat unchanged.
  - **⚠ On the chat endpoint, adding a `filter` silently unscopes the query.** The route passes
    `session_id=None if line_filters else session_id`, so a line filter *replaces* the session scoping
    instead of narrowing it. Verified live: `filter=|=:.` returned **4239 rows of which only 837 belonged
    to that session** — you would think you narrowed and instead widened 5×. On chat, carry the id in the
    filter chain yourself (`?filter=|=:<sessionId>&filter=|~:ERROR`); `level=` and `q=` are safe, and the
    voice endpoint is unaffected because it applies `session_id` unconditionally.
  - **⚠ Corrected my own advice: do not scope logs by label.** The earlier text suggested narrowing via
    `/logs/labels`. Two verified failures: `label=` alone returns `400 "No search criteria"` (label filters
    don't satisfy the criteria gate — you still need `session_id`/`filter`/`q`), and the labels list is
    **cluster-wide, not call-scoped**, so a plausible label silently matches nothing — scoping by
    `request_id` returned **0 rows** for a call that has 763, because its `/values` holds 934 UUIDs
    belonging to other services while voice-ai carries its request id in the line body. `session_id`
    (line-contains, the call's `requestId`) is the only reliable scoping.
  - **The resolved config is in the logs at DEBUG — and it is the answer to "what did this call actually
    run with".** Ask for it by name and it is cheap; crawling for it is not (`level=debug` alone is 1488
    rows / 985 KB). `filter=|~:Raw Config|Worker Config|Request Config` → 3 rows / 61 KB (the worker JSON
    as fetched *and* as composed); `|~:system instruction` → **1 row / 4.7 KB, the composed system prompt
    exactly as the model received it**, which is ground truth for any prompt defect;
    `|~:tools configured|No .* configured` → 5 rows / 2.7 KB, an explicit inventory of what was *not*
    wired (`No custom code tools configured`, `No A2A agents configured`, …). **That inventory is the
    earliest possible catch for a dangling `[tool:…]` reference** — logged at call setup, ~10 seconds
    before the LLM attempts the call and fails. Also present: auto-injected channel variables
    (`language CHANNEL variable: default=en, allowed=['en']`) and prompt/topology validation verdicts.
  - So the documented approach is now an **escalation ladder**: `level=error` (3 rows / 2.3 KB) →
    `level=warning` (4 / 2.8 KB) → `level=info` (21 / 13 KB — transcript turns, latency, session
    lifecycle) → **targeted** `level=debug` with a `filter`. All four levels work, despite
    `/logs/labels/level/values` listing only `error` and `info` — one more reason not to trust that
    endpoint.
  - **⚠ `createdAt` is not the call start, and the old window advice was wrong.** Verified live: a call
    with `createdAt 06:07:56` / `duration 39.78s` had log activity from **06:07:14 to 06:08:34** — 42 s of
    it *before* `createdAt`, including the whole config load. The previously suggested
    `[createdAt − 30s, createdAt + duration + 30s]` would have missed every config line. Now
    `[createdAt − duration − 90s, createdAt + 90s]`; Call Analyzer hedges about this itself via
    `anchorSource: "created_at_minus_duration"` and a `heuristic-transcript-anchor` warning.
  - Also captured: the modules worth filtering on (`pipeline_events`, `llm_switcher`, `llm_log_observer`,
    `call_manager` for `Transcript [role]:` lines, `pipeline_factories`, `worker_builder`,
    `session_tracker`) versus the noise (`unified_serde` and `text_transformer` are a third of all lines);
    that not every alarming DEBUG line is a fault (`get_session_info … status=404 "Session not found"` is
    immediately resolved by `SUCCESS (empty) | reason=no_messages` — read the resolution before reporting
    a 404); and **a call's logs survive the worker being deleted**, so post-mortems stay possible after
    cleanup.
- **Call Analyzer is a shared plane, not a debug-only one — every skill that reads a call can now read
  it.** The first cut scoped `commotion_analyzer` to `commotion-debug` and mentioned it in the loop only
  as a fallback when the evaluator errored. That was too narrow: **a simulation is just a real call with a
  robot caller**, so every signal on that plane applies to a scenario-run exactly as it does to production
  traffic — and `commotion-improve-worker`'s own taxonomy already conceded that some failures ("a
  mispronunciation surfaces as the tester bot mis-hearing a term") are visible *only* in the transcript.
  So `mcp__commotion__commotion_analyzer` is now in `allowed-tools` for `commotion-run-evals`,
  `commotion-improve-worker`, `commotion-generate-scenarios` and `commotion-create-worker`, each with an
  active use rather than a fallback:
  - **`commotion-run-evals`** — Phase 4 gains a step that pulls each failing run's actual call
    (`/api/calls?requestId=<scenario-run-id>` → `?fields=transcript,metrics,analysis`) and folds four
    things the eval surface cannot answer into the "why" column: did the run happen at all
    (`userTurnCount`), did the tools fire and what came back (`toolCallMetrics[].result`), was "slow" the
    tool or the model, and was it the worker's fault at all (`contextCorruption`, `llmFallbackEvents`,
    `state_load_errors`). This is what makes the handoff actionable — *"failed the goal"* sends the improve
    loop guessing at the prompt; *"called `get_policy_status`, got `not registered`, then asserted the
    answer anyway"* names the fix.
  - **`commotion-improve-worker`** — the failure→fix taxonomy gains a **"what proves it"** column keyed to
    Call Analyzer signals, plus three new rows it could not previously diagnose (mispronunciation, "too
    slow" split into tool-vs-model, talked-over-the-caller) and a final **"not the worker at all"** row
    pointing at the platform-artifact table. Diagnoses must now quote the evidence, not paraphrase it.
  - **`commotion-generate-scenarios`** — Phase 3c's "from a real call" path now says how to *find* that
    call (`/api/calls?workerId=`, `/api/reports/latest`) and to copy the caller's turns **verbatim**,
    since STT artifacts are frequently the trigger. It is the highest-value scenario source available
    because it is drawn from real behaviour rather than imagination.
  - **`commotion-create-worker`** — Phase 12's text spot-check now reads itself back as a `COPILOT` session
    (`/api/chat/session/<id>`), which is the only way to see whether the tools wired in Phase 8 actually
    fired. A `[tool:<name>]` with no match in `registered_tools` is a dangling reference that returns
    `"Error: function '<name>' is not registered."` and makes the agent fabricate — the #1 failure that
    phase already warns about, invisible in the reply text and obvious here.
  - **`commotion-quality-loop`** — reframed: the specialists are expected to hand back evidence from the
    calls, and a round returning bare "failed the goal" reasons should be challenged before another round
    is spent on a guess. The orchestrator itself stays thin — it deliberately does **not** take the tool.
  - `call-analyzer-api.md` is retitled the **observability plane** and declares itself the canonical map
    for it (the same idiom `eval-domain-api.md` uses), with an explicit note that outside
    `commotion-debug` the tool is **optional enrichment**: if it isn't registered, say so once and fall
    back to `evaluationReasoning` — never block a run on it.
- **The simulation/eval findings are propagated to every skill that reads a pass-rate**, because they
  affect the existing quality loop at least as much as the new skill — a `passRate` that isn't measuring
  the worker breaks a threshold loop worse than it breaks a one-off RCA.
  - **`commotion-run-evals`** — Phase 3 gains a gate requiring `passRate` to be reported *with how many
    runs were decided* (a bare `0.0` from unevaluated runs is the misleading case). Phase 4's breakdown
    becomes a **three-way bucketing** — pass / fail / **not evaluated** — with the not-evaluated bucket
    excluded from the rate and passed downstream *labelled as such*, never as a failure to diagnose.
    `references/simulation-and-results.md` documents the `ERROR`/`''` values, `scenarioRunStatusLabel`,
    the extra `failuresReasoning[]`/`passesReasoning[]` fields, the ambiguity of the 1.4.1 signature, and
    that a `500` on trigger is not always transient.
  - **`commotion-improve-worker`** — Phase 1 must separate "failed" from "never evaluated" *before*
    diagnosing, since a round spent on an unevaluated or mute-caller run moves a prompt at random.
    `references/improvement-loop.md` gains a "the pass-rate must be measuring the worker" section:
    compare only like denominators, and treat zero decided runs as a **blocked loop**, not a failure.
  - **`commotion-quality-loop`** — Phase 3 gains a hard gate: require the decided-run count from
    run-evals and **do not enter the improve loop when nothing was decided**, because iterating against a
    score that cannot move burns every round. Phase 4's first branch stops on an unmeasured baseline. The
    loop still sequences exactly the four specialists; `commotion-debug` appears only as a pointer inside
    that caveat, explicitly marked as not part of the loop.
  - **`commotion-generate-scenarios`** — Phase 2's "reuse an existing personality" advice now carries the
    `voiceEnabled` check, since all ten dev3 personalities had it `false`; and
    `references/scenarios-and-personalities.md` records that **`aiAgentId`, not `version`, is what binds a
    scenario to a runnable target**, that a stale `aiAgentId` is the real cause of the `500` on trigger,
    and that `PUT /scenario` silently ignores a changed `version`.
- **Chat is honest about its weaker gate.** Simulations are voice-only, so a chat session gets full RCA
  and the same edit mechanics but verifies via a `POST /aiworker/run` text spot-check, reported as
  `repro: n/a (chat — text spot-check only)`. A text spot-check is not a reproduction and the skill says
  so rather than implying the confidence a 3-of-4 simulation would carry.
- **⚠ Security note the operator must resolve.** The Call Analyzer api key resolves at Kong to persona
  `admin` with `allowed_orgs: []` / `allowed_workspaces: []` — unrestricted cross-tenant reads, including
  unmasked transcripts and composed system prompts. Call Analyzer's own trust model treats `admin` as
  internal-only, so tenant scoping is an open decision isolated in one seam on the MCP side
  (`analyzer_shaping.enforce_scope`); the plane is meanwhile gated only by whether the key is configured.
  Do not enable it beyond dev3 until that lands. Bumped to 1.5.0.

## 2026-07-27 — 1.4.1 — connector credentials are create-time only; cap simulations at 4 runs

Two defects found live: a worker whose Gmail/Calendar connectors silently never registered, and
simulations that "completed" instantly with `passRate 0.0` when too many scenario-runs were submitted at
once. Both were traced to skill wording that taught the wrong thing; no new endpoints.

- **A connector must carry its credential in the create body**
  (`commotion-create-worker/SKILL.md` Phase 6 + `references/tools-and-capabilities.md`). The skills said
  `credentialMetaDataInput` was *optional* — "attach the actions now, `PUT` the credential later". That
  is wrong: without a connected `credentialIdentifier` the `POST /ai-worker-tool/connector` returns `200`
  but the backing managed MCP server never registers (empty `actionName`/`inputSchema`/`outputSchema`, no
  `credentialMetaDataOutput`, and the UI banner *"Tools could not be registered — MCP server failed to
  register, or the MCP authentication token is missing."*). And there is **no** `PUT` path to fix it —
  `UpdateConnectorToolRequest` has no credential field — so a bare connector can only be deleted and
  re-created. The skills now treat the credential as functionally required, tell you to reuse a
  `"connected": true` credential from `GET /ai-worker-tool/credentials`, and never to attach a connector
  before one exists.
- **Display metadata is POST-only too.** A minimal create registers but renders bare in the UI (generic
  "G" icon, empty **Category**, no action chips). The connector recipe now threads `appIconUrl` +
  `appTags` (from `/integration-apps`) and per-action `actionDisplayName` (from `/app-actions`) through
  the same create body. `actionDescription` is accepted but not persisted — don't rely on it.
- **Cap each simulation at 4 total scenario-runs** (`commotion-run-evals/SKILL.md` Phase 2 + guardrails,
  `references/simulation-and-results.md`, `commotion-improve-worker/SKILL.md` Phase 3). A sim runs its
  scenario-runs concurrently over websockets; more than 4 at once exhausts the connections and the excess
  runs fail with connection errors — surfacing as the existing failure signature (`COMPLETED` instantly,
  `passRate 0.0`, `avgLatencyInMillis null`). `maxScenarioRunLimit` (20) is no longer the practical
  ceiling: keep `sum(scenarioIdToRunPerScenarioMap.values()) ≤ 4` per sim and run larger sets as
  sequential batches of ≤4 (poll each to completion, confirm `/scenario-run/active` is clear, then start
  the next), aggregating pass-rates at the end.

## 2026-07-22 — 1.4.0 — browser-login OAuth is live; interim in-chat login removed

The Commotion backend shipped its full OAuth 2.1 authorization server, so auth is now the automatic
**browser login** the skills always pointed toward — the interim "ask for email + password →
`commotion_login`" stopgap is gone. On first use of the Commotion MCP the client opens a Commotion
login in the browser, the user signs in once, and the token is attached to every call automatically;
the raw credential/token never enters the conversation.

- **Skills no longer collect credentials.** All five `SKILL.md` files drop
  `mcp__commotion__commotion_login` from `allowed-tools` and replace the "sign in first (interim)"
  preamble with the automatic-OAuth wording (never ask for an email/password or a token, never pass a
  `token` argument). `commotion-create-worker/references/api-and-auth.md`'s Auth section is rewritten
  from interim in-session login to the OAuth browser flow, and the optional `token` note is removed.
- **MCP server flipped to the Path B end-state** (`commotion-mcp` repo). `interim_mode` is now off:
  `/mcp` is an OAuth resource server that emits the RFC 9728 `401` challenge (so the client opens the
  browser), advertises the BE authorization server
  (`https://auth-tier0.dev3.gocommotion.com/auth/oauth/mcp`) in
  `/.well-known/oauth-protected-resource`, and forwards the bearer to Kong. The token verifier is now
  **decode-and-forward** — the BE issues HS256 JWTs with no JWKS, so it enforces `exp` and forwards
  the token (Kong/BE are the real authorization gate) rather than verifying an RS256/JWKS signature.

## 2026-07-22 — 1.3.0 — migrate to the redesigned Guardrails API + new safety layers

The backend shipped a redesigned Guardrails schema (dev3 Swagger `/v3/api-docs/public`). Guardrails
still live on `AiWorkerRequest.guardrailConfigRequest` set via `POST`/`PUT /aiworker` — transport, the
draft gate, and the inbound-vs-outbound direction philosophy are unchanged — but the nested shape
changed in breaking ways and gained new safety features. The skills documented the old shape, so they
would have built invalid/outdated payloads. All guardrail-facing skill text is updated to the new
schema and the two new capabilities are now taught.

- **Breaking schema fixes** (`commotion-create-worker/references/control-and-reliability.md` + Phase 3):
  - **Toxicity** — `toxicityDetectionMethod:"LLM_BASED_DETECTION"` → `toxicityDetectionModel:"QWEN3_GUARD"`;
    float `toxicityThresholds` → per-category boolean `toxicityCategories`; `actionOnToxicityDetection`
    removed.
  - **Forbidden words** — flat `forbiddenWordsConfigRequestList` → grouped
    `forbiddenWordGroupsConfigRequest:{enabled, wordGroups:[{words, fallbackMessage, endCallOnDetection}]}`.
  - **PII** — `piiByCommotionEnabled`/`piiByCommotionConfigList` → `builtInPiiCategoryConfigRequest`
    (`enabled` + `builtInCategories:[{category, action}]`); regex items gain `maskingRegex`.
  - **Custom checks** — added `inboundEnabled`/`outboundEnabled`/`isVoiceEnabled` toggles + item-level
    `endCallOnDetection`.
- **New safety features now taught**:
  - **`advancedSafetyConfigRequest`** — `manipulationDetectionEnabled` (first-class prompt-injection /
    jailbreak / social-engineering defense) and `focusGuardrailEnabled` (anti-drift / scope re-alignment).
    Wired into Phase 3's interview, the improve-worker symptom→fix table, and the run-evals signal list.
  - **Voice-only controls** — `executionMode` (`STREAMING`/`BLOCKING`), `endCallOnDetection` across
    toxicity / forbidden groups / custom rules, and audio masking for call recordings
    (`audioMaskingConfigRequest`).
- **Scenario coverage** (`commotion-generate-scenarios/SKILL.md`) — adds prompt-injection /
  social-engineering and scope-drift scenarios so the new advanced-safety layers get exercised.
- The category names for `toxicityCategories` and `builtInCategories[].category` remain **dynamic** —
  the skills continue to ground them in `GET /aiworker/metadata`, not hard-coded enums.
- **Verified live** — round-tripped all six blocks on chat + voice draft workers, and drove a deployed
  worker through `POST /aiworker/run` against a no-guardrail control. Findings now baked into the skills:
  the blocking guardrails (toxicity, forbidden-word groups, inbound custom, manipulation detection) do
  **intercept** the tripping input (control worker succeeded on the same inputs); however a block
  currently returns `status:"FAILED"` with a generic *"reference number"* error instead of the
  configured fallback text — a **known code-side/backend bug**, not the guardrail contract, so the skills
  flag it as such and steer guardrail-UX verification to the delivered channel rather than the sync run
  API. PII masking is **not** applied on the sync text run API either; `regexPatternEnabled` is required
  whenever `piiMaskingConfigRequest` is present. All test workers were deleted afterwards. Guardrail
  **endpoints/auth are unchanged** (still `POST`/`PUT /aiworker`), so `api-and-auth.md` needed no
  guardrail change — but its `/aiworker/run` note was corrected with two runtime requirements found while
  testing: a run needs an identity (`userId`/`fingerprintId`/`audienceId`) and a deployed worker with an
  enabled agent.

## 2026-07-22 — 1.2.1 — close production-worker gaps: state vars, quality-loop handoff, guardrail direction

Three gaps surfaced from a real production build (a flight-booking worker) where the model built the
worker/agents/tools/prompt correctly but (a) never added **state variables**, (b) stopped at build
without running the **sim → eval → improve** loop, and (c) configured mostly **outbound** guardrails
when **inbound** is what a transactional worker needs. All three were traced to skill wording that let
those steps be skipped; no new endpoints.

- **State variables are decided during the interview, not forgotten at the end**
  (`commotion-create-worker/SKILL.md`). Phase 1's capability checklist now explicitly asks whether the
  worker must remember caller-provided values (so it doesn't re-ask) or pre-load profile/account/booking
  data — with booking/renewal/lookup flagged as almost-always-yes. Phase 8.5 gained a "state it out
  loud before moving on" forcing line, and the execution-rules note now separates 8.5's two halves:
  **state variables are frequently *necessary***, while **pronunciation** stays the discovered-later one.
- **A build no longer dead-ends before it's proven.** `commotion-quality-loop`'s description now names
  *production / production-ready worker / long detailed problem statement* as its default entry point,
  so a production statement routes to build+test+improve instead of build-only. `commotion-create-worker`
  gained the **`Skill`** tool and its Phase 12 is rewritten from a soft "note" into a proactive
  `AskUserQuestion` that offers to hand the built worker to `commotion-quality-loop` (which handles the
  deployed-voice prerequisite) — for a production use case, testing is mandatory, not a footnote.
- **Inbound vs outbound guardrails are now taught, with the correct default**
  (`commotion-create-worker/SKILL.md` Phase 3 + `references/control-and-reliability.md`). `inbound`
  filters the caller's message; `outbound` filters the model's own output. The model already has
  provider-side safety training, so **default to inbound-only for toxicity** and enable outbound only
  when the model's *own text* is the risk (PII echo-back, competitor/confidential terms it might say, a
  "never say/advise X" rule). Custom checks are directed to the right side ("don't accept X" → inbound;
  "never say X" → outbound); PII masking inherently covers both. The reference calls out the exact
  anti-pattern (outbound-heavy config on a transactional worker) and the recipe now shows inbound-only
  by default.

## 2026-07-21 — 1.2.0 (refinements) — use-case-driven settings, code-block state vars, prompt-write correctness

A review pass on the 1.2.0 Settings work. No new endpoints — sharper judgment about *when* to reach for
each feature (so the agent prompt stays clean), a code-block ↔ state-variable link that couldn't be wired
before the variable API existed, a prompt-update mechanics correction, and verified-live updates.

- **Feature selection is use-case-driven, not blanket-optional** (`commotion-create-worker/SKILL.md`).
  Phase 8.5 reframed: **state variables** are frequently *necessary* (remember caller data / don't
  re-ask / pre-load profile) — defining one keeps that out of the prompt instead of bloating it;
  **pronunciation dictionaries** stay optional and are usually *discovered from a simulation run* rather
  than pre-guessed. The execution-rules note now frames Phases 7/8/8.5 as "run when the goal calls for
  it," and ties guardrails/forbidden-words to the same design-then-tune-from-sims discipline.
- **Simulation runs as a diagnostic signal.** `commotion-run-evals/references/simulation-and-results.md`
  gains a "reading a run for settings signals" section — a mispronunciation shows in the **transcript**
  as a tester-bot ASR mismatch (not in the score); re-asked/forgotten data → a state variable; an
  off-limits or over-blocked answer → a guardrail tune. `improvement-loop.md` notes the transcript cue
  for mispronunciation alongside `evaluationReasoning`.
- **Code-block tools ↔ state variables** (`tools-and-capabilities.md`). Documented that a code-block's
  `stateVariables[].id` is the variable's **`id`** from `POST /ai-worker-variable-schema` (create it in
  Phase 8.5 first), that the UI equivalent is the **`@variable`** insert, and that one variable is
  consumable two ways — the prompt token `[var:<title>]` and a code-block `stateVariables[]` entry.
- **Prompt-write correctness (POST vs PUT).** Every *newer* spot that bound a `[var:]`/`[tool:]`/
  `[knowledge:]` token via a bare `PUT /aiagent/{id}` now points at the canonical Phase-6 rule: compose
  the token into `instructions` and **(re-)`POST`** so the prompt renders in the UI — a bare `PUT` writes
  the runtime only and leaves the editor stale. **Verified live 2026-07-21:** a `PUT` on the
  auto-provisioned default left the UI prompt editor **blank**; a fresh `POST` **rendered** the prompt
  (with `[var:]`/`[tool:]` tokens shown as plain text, not chips). Corrected two agent types too:
  **structured-output** now uses the same **delete-default + `POST`** flow (verified `POST /aiagent`
  accepts `STRUCTURED_OUTPUT` → 200) so its prompt renders; **FAQ** is the sole exception — `POST /aiagent`
  rejects FAQ types and `/aiagent/standard` has no `instructions` field, so FAQ instructions are
  `PUT`-only and never render in the UI editor (edit in the UI for visibility). The single-agent
  "common case" recipe was also switched from PUT-enabling the default to delete-default + POST.
- **Verified-live updates** (`settings-variables-pronunciation.md`): the TTS **applies** a pronunciation
  entry and an **`EXTRACTED`** variable **is populated** from a real call — both verified;
  `LOADED`-fetch stays not-yet-verified. Live A/B on dev3 also confirmed: **code-block ↔ state variable**
  (`stateVariables:[{id}]` binding + `/code-block/run` substitution, where the placeholder injects a
  **pre-quoted literal** — `x = {{[statevar:title]}}`, not `"…"`), and that state variables /
  pronunciation entries render under **Settings** in the UI.

## 2026-07-20 — 1.2.0 — Settings: pronunciation dictionaries + state variables (and eval-domain reconcile)

The backend shipped worker **Settings** APIs the skills didn't cover. Added them to
`commotion-create-worker` (they're worker-definition config, so they extend that skill rather than a
new one) and reconciled the eval-domain endpoint map against the live spec. All grounded in the live
OpenAPI spec (`/v3/api-docs/public`, 215 paths) and verified with real CRUD against a throwaway voice
worker on dev3 (`6a5dce0cff0a5eea9a86c6c9`, draft v0).

- **New reference `settings-variables-pronunciation.md`** (under `commotion-create-worker/references/`)
  + **Phase 8.5** in the SKILL, covering two worker-scoped resources (full CRUD; created on a draft,
  `(worker, version)`-scoped — **not** fields on `AiWorkerRequest`):
  - **Pronunciation dictionaries** (`/ai-pronunciation-dict`, `AiPronunciationDictRequest`) — teach the
    TTS domain terms. `pronunciationType` ∈ `ALIAS`/`IPA`/`CMU`/`SYMBOL`/`PHONEME`; `inputText` unique
    per worker+version. **Gotcha (verified live):** the id comes back as **`pronunciationDictId`**, not
    `id`; a duplicate `inputText` → `400 "… already exists."`
  - **State variables** (`/ai-worker-variable-schema`, `AiWorkerVariableSchemaRequest`) — values tracked
    across a call. `variableSource` `EXTRACTED` (LLM pulls from the conversation) vs `LOADED` (fetched
    from a tool). **Gotchas (verified live):** id comes back as **`id`**; a `LOADED` variable **requires
    `loadingStrategy`** (`400 "Loading strategy is required when variable source is LOADED"`) + a
    `toolReference`; creating a variable does **not** bind it — an agent must reference `[var:<title>]`
    in its prompt (this replaces the old "state appears on its own" note in Phase 8).
- **Knowledge settings** (`/aiworker/km-setting`, `KMSettingUpdateRequest`) documented in
  `knowledge-and-rag.md`: an **auto-provisioned** indexing/embedding/chunking config edited by id —
  `GET /aiworker/km-setting/{workerId}` → read `settingId` → `PUT …/setting/{settingId}`. (This is what
  `KMSettingUpdateRequest` actually is — Knowledge Settings, **not** conversation analysis, which has no
  dedicated endpoint in the current spec.)
- **Eval-domain reconcile** (`eval-domain-api.md`): added the newly-shipped surfaces —
  `/eval-insight-group` (+ `/refresh`, `EvalInsightGroupRequest`; grouped failure-mode analysis),
  `/eval-result/count` and the filterable `/eval-result` list, and `/schedule` (`ScheduleConfigInput`,
  delayed-webhook scheduler). Marked as secondary/analysis surfaces — the loop's gate is still the
  scenario `passRate`. (`PersonalityRequest`'s voice/noise fields — `speakingSpeed`,
  `interruptionLevel`, `backgroundNoise*`, `packetLoss` — were already documented in
  `scenarios-and-personalities.md`, so no change there.)
- **`improvement-loop.md`** failure→fix taxonomy gained two rows: re-asks for known data → define a
  **state variable**; mispronounces a brand/acronym → add a **pronunciation dictionary** entry.
- Endpoint map + `commotion_schema` name list in `api-and-auth.md` extended with the three new families
  (`AiPronunciationDictRequest`, `AiWorkerVariableSchemaRequest`, `KMSettingUpdateRequest`). Bumped
  `plugin.json` / `marketplace.json` to 1.2.0.
- **Not yet verified (need a live conversation, not config round-trip):** that the TTS applies a
  pronunciation entry at runtime, that an `EXTRACTED` variable is populated from a real call, and that a
  `LOADED` variable fetches from its tool. Noted in the new reference's "verified live" block.

## 2026-07-20 — 1.1.0 — Interim in-session login (until the Commotion browser login ships)

Each skill now **signs in in-session** before calling the platform: it asks for the user's Commotion
email + password (`AskUserQuestion`), calls the new **`commotion_login`** MCP tool to get a per-user
access token, and passes that token on every `commotion_request` / `commotion_schema` call — so dev3
attributes actions to the signed-in user. This is a stopgap until the Commotion **browser login**
ships (then auth becomes automatic again, no in-chat credentials). Adds
`mcp__commotion__commotion_login` to each skill's `allowed-tools`; the full flow lives in
`commotion-create-worker/references/api-and-auth.md`.

## 2026-07-10 — 1.0.0 — Transport moves to the thin Commotion MCP server (OAuth)

The skills no longer call dev3 with local scripts and a Kong api-key. All platform I/O now goes
through the **thin Commotion MCP** server (repo `commotion-mcp`), which holds the credential and
proxies each call. Two tools replace the old `scripts/`:

- **`commotion_request`** — one authenticated call: `{ method, path, body? }` → `{ status, body }`
  (a non-2xx is **returned, not thrown**; `path` is relative — the base URL is fixed server-side; the
  request payload is the `body` argument, so there are no temp files). Replaces `commotion_api.sh`.
- **`commotion_schema`** — a bundled request schema by component name: `{ schema_name, refresh? }`.
  Replaces `fetch_schema.sh` (+ `bundle_schema.py`); the spec is bundled server-side.

- **Auth is OAuth, owned by the MCP client — no API key, no "Step 0".** On first use the client opens
  a Commotion login in the browser and attaches the user's token to every call automatically; the raw
  token never enters the conversation. If the tools aren't connected, authorize via `/mcp` →
  **commotion** → Authenticate.
- **`plugin.json`** now registers the MCP server (`mcpServers.commotion`, a hosted HTTP endpoint).
  **Removed** `scripts/` (`commotion_api.sh`, `fetch_schema.sh`, `bundle_schema.py`, `scripts/README.md`)
  and `.env.example` — nothing on the client to configure.
- **All five skills** rewritten: each skill's "Step 0 / Kong key" preamble is gone; every call site
  now uses the two tools; id-threading reads the returned `body` (no `jq`) and poll loops are
  model-driven (no `bash`/`sleep`). `allowed-tools` updated (MCP tools added; `Bash` kept only in
  `create-worker` for the direct Azure Blob presigned-URL byte-PUT, which isn't a dev3 call; `Skill`
  kept in the quality-loop orchestrator). Endpoint maps, schema names, and all behavioural guidance
  are unchanged — only the transport + auth changed.
- **Verified live end-to-end (2026-07-10):** minted an OAuth token through the login flow and drove
  the two MCP tools against dev3 — `commotion_schema`, `GET /aimodel`, **create worker**, **add agent**
  (list → delete default → POST new), and **add a code-block tool** — all succeeded; the eval-domain
  read endpoints (`/scenario/dropdown-config`, `/scenario/intent-values`, `/eval-metric`,
  `/personality`) are reachable too. Confirmed the proxy returns a non-2xx as `{status, body}` rather
  than throwing. New gotcha captured: **worker names are globally unique** — a duplicate `POST
  /aiworker` returns `400 "a worker with the name 'X' already exists"` (now noted in create-worker
  Phase 5).

## 2026-07-07 — 0.6.0 — Add the `code-block` tool kind (sandboxed Python)

dev3 shipped a new AI-worker tool kind — **`code-block`**, custom **Python** run in a sandbox. The
skills previously stated the opposite ("the custom tool is an HTTP wrapper — there is no code mode"),
which is now false. Documented the kind end-to-end and corrected the stale negations, all grounded in
the live spec and a real dev3 probe (created + listed + test-ran + cleaned up a throwaway worker).

- **New endpoints** (in `references/tools-and-capabilities.md` + `references/api-and-auth.md`):
  `POST/PUT /ai-worker-tool/code-block[/{id}]` (create/update, schema `CreateCodeBlockToolRequest`) and
  `POST /ai-worker-tool/code-block/run` (stateless sandbox test-run, schema `RunCodeBlockRequest`).
- **`codeBlockMetadata`:** required `name` (unique in version), `description`, `language` (`PYTHON`
  only), `sourceCode`; optional `hitlMode` (this kind **does** support HITL), `contextPrompt`,
  `outputAsJson`, `toolUsageTypes` (`LLM`/`PRE_LOAD`), and variable refs (`stateVariables[]`,
  `llmVariableIds[]`, `systemVariableIds[]`) referenced in source via `{{[statevar:..]}}` /
  `{{[llmvar:..]}}` / `{{[sysvar:..]}}` placeholders (distinct from the prompt `[var:..]` token).
- **Binding (verified live):** for `CODE_BLOCK`, `actionMetaDataOutputList` is **empty** — the bindable
  name is `codeBlockMetadataOutput.lowerCaseName` (your `name`, used as-is, no numeric suffix). Bind
  with `[tool:<name>]` in the agent's `instructions`.
- **Sandbox limits kept prominent:** no network, no filesystem — external calls still go through a
  `custom-tool`. Updated golden rules 4 (HTTP-vs-code) and 6 (HITL now includes code-block), the
  "picking the kind" table, the per-kind body + recipe, the binding section, and Phase 8 of the SKILL.
- Added `CreateCodeBlockToolRequest` / `RunCodeBlockRequest` to the `fetch_schema.sh` name hints.
  Other skills (`improve-worker` reuses this reference; scenarios/evals/quality-loop) inherit it.
  Bumped to 0.6.0.

## 2026-07-02 — 0.5.0 — Add the `commotion-quality-loop` orchestrator (single entry point)

The four skills were independent — chaining relied on the model following the "step N of the loop"
prose. Added a thin **coordinator** skill so the whole pipeline runs from one request, with the
iterate-until-threshold control flow owned in one place.

- **`commotion-quality-loop`** — the end-to-end orchestrator. Triggers on compound requests ("build a
  voice bot for X and make it pass 90%", "test and improve my worker until 80%") and **invokes the four
  specialists in order via the `Skill` tool** (`allowed-tools` now includes `Skill`): ensure a deployed
  **voice** worker (build via create-worker if needed) → generate-scenarios → run-evals (baseline) →
  improve-worker (loops on a draft until `passRate ≥ threshold` or max rounds) → deploy on approval. It
  carries shared state (worker id, version, scenario ids, SIM_ID, pass-rate, threshold, max rounds)
  between steps and owns the threshold/max-rounds/deploy gates; it does **not** duplicate the
  specialists' internals.
- **Routing:** compound "whole loop" requests → the coordinator; single-step requests → the specialist
  directly. Added a "for the full loop, use `commotion-quality-loop`" pointer to each specialist's
  "When to use this", and listed the coordinator as the entry point in the README.
- Enforces the live-verified prerequisite up front (a **deployed voice** worker) so the loop doesn't
  start against a chat/never-deployed worker that can't be simulated. Bumped to 0.5.0.

## 2026-07-01 — 0.4.1 — Live-test hardening of the quality-loop skills (dev3)

Dogfooded the full loop end-to-end on a real dev3 worker (`Acme Support Triage`): built it, generated
personas + scenarios, ran voice simulations, drove an improve round (ETA-gap scenario `0%→100%` after
a prompt edit), added Hindi/English language switching, and populated eval-metrics. Folded every
backend behavior the live run surfaced into the skills.

- **Evals/simulations are VOICE-ONLY, and need a deployed worker.** A chat worker fails every sim run
  ("An error has occurred during simulation"); a **never-deployed** worker returns *"Worker is not
  available"* and AI scenario-gen yields nothing. A **draft version of an already-live worker CAN be
  simulated** — which is what makes the draft-only improve loop actually work. Added prerequisites to
  run-evals, generate-scenarios, and improve-worker; noted in create-worker Phase 12.
- **`passRate` is a percentage 0–100** (not 0–1). Default loop threshold corrected to **80**.
  `SimulationResponse.avgQuality` stays `null` (not wired to eval-metrics).
- **Eval-metric create recipes (the sharp part).** *Custom* = full body (`metricSourceType:"CUSTOM"` +
  criteria/threshold). *Standard/predefined* = **POST a minimal shell → PUT the full definition,
  dropping empty-string fields** (an empty `evaluationMethod` 500s; full-body POST 500s on `name`;
  minimal POST alone = a hollow shell). `version` must be the **live** version. Fetch the catalog via
  `GET /eval-metric?metricSourceType=STANDARD`. Verified Hallucination/Relevancy/Appropriate Call
  Termination/CSAT/Sentiment all hydrate this way. Rewrote `run-evals/references/eval-metrics.md`.
- **Two evaluation surfaces.** Simulation scenario pass/fail (Simulations → Runs) ≠ the **Evals
  dashboard** (eval-metric results). The dashboard is empty unless metrics exist *and* their evaluation
  runs — which is **async**: sim calls create eval-results in `status: PENDING`; force scoring with
  `POST /eval-result/trigger?voiceCallId=<voiceInteractionId>` (the **`voiceInteractionId`**, not
  `voiceCallMongoId`); a scenario-run's `id` **is** the call's `sessionId`. Rewrote
  `run-evals/references/simulation-and-results.md`.
- **Agent type is immutable via PUT** — change it by delete + re-POST (added to
  `create-worker/references/agents-and-orchestration.md`).
- **Language switching (en + hi) pattern**, validated live: worker
  `workerVoiceConfiguration.allowedLanguages:["en","hi"]` + an agent prompt rule (mirror the caller's
  language; **don't** switch on English-spelled numbers/emails). Documented in generate-scenarios
  (bilingual personas) and improve-worker (a config-level fix example).
- **Misc verified:** `/aiworker/{id}/versions` returns `{"items":[…]}`, superseded status is
  **PAUSED**; scenario/metric list bodies can contain **raw newlines** (parse tolerantly);
  dropdown-config codes (SIMPLE/MODERATE/COMPLEX, HAPPY/JAILBREAK, VOICE/CHAT, limits 20/20); AI
  scenario-gen is async with **no progress endpoint** and needs a live worker (manual `POST /scenario`
  is the reliable fallback). Bumped to 0.4.1.

## 2026-06-30 — 0.4.0 — Close the quality loop: generate-scenarios, run-evals, improve-worker

Three new skills extend the plugin from "build a worker" to "build, **test, evaluate, and iteratively
improve** a worker until it clears an eval-score threshold" — the loop create-worker →
generate-scenarios → run-evals → improve-worker.

- **`commotion-generate-scenarios`** — builds a worker's test set: simulated-caller **personalities**
  (`/personality`, with AI-drafted prompts) and **scenarios** (`/scenario`) — AI-generated (async
  `POST /scenario/generate` → poll), manual, or from a real call. References: `eval-domain-api.md`
  (the canonical endpoint map for the scenario/sim/eval domain) + `scenarios-and-personalities.md`.
- **`commotion-run-evals`** — optionally defines **eval metrics** (`/eval-metric`), runs the scenarios
  as a **simulation** (`POST /simulation/run`), polls `GET /simulation/{id}`, and reports the
  **pass-rate** + per-scenario failures (`GET /scenario-run?simulationId=`). References:
  `eval-metrics.md` + `simulation-and-results.md`.
- **`commotion-improve-worker`** — owns the **iterate-until-threshold loop**: diagnoses failing
  scenarios (`failureReason`/`evaluationReasoning`), edits the worker on a **draft** (reusing
  create-worker's machinery), re-runs the evals, and repeats until `passRate ≥ threshold` or a round
  cap — then deploys on approval. Reference: `improvement-loop.md` (loop control, regression guard,
  version-pinning, failure→fix taxonomy).
- **Locked design decisions** (from build session): the improvement loop is **draft-only** (never
  auto-deploys mid-loop; the live worker is untouched until the user approves the final version), and
  the gate is the **scenario pass-rate** (`SimulationResponse.passRate`; default target 0.8, default
  max 3 rounds, both asked at runtime).
- **Transport reuse — no new scripts.** Verified against the live OpenAPI spec
  (`/v3/api-docs/public`, 193 paths) that the scenario/simulation/eval endpoints are part of the
  **same unified backend** as `/aiworker`/`/aiagent`, so all three skills reuse `commotion_api.sh` +
  `fetch_schema.sh` + the Step-0 Kong-key flow unchanged. Each skill smoke-tests the eval route
  (`GET /scenario/dropdown-config`) first.
- **Open items flagged for live testing** (folded into the references): eval-domain route parity
  through Kong; version carry-over of scenarios/metrics across a new draft; `standardEvalMetricId`
  predefined-metric catalog; scenario-run → session/call id linkage for per-metric scores; async
  scenario-generation poll shape; simulating a draft end-to-end. There is **no server-side
  "improve prompt" endpoint** — improvement is model reasoning + the create-worker editing machinery.
- Bumped `plugin.json` / `marketplace.json` to 0.4.0 and widened the descriptions. Skills are
  auto-discovered from `skills/*/SKILL.md` (no manifest enumeration needed).

## 2026-06-29 — 0.3.6 — Prompt for the Kong api-key at session start (no committed/embedded key)

The skill no longer needs a committed `.env` or an embedded secret — it asks the user for the Kong
api-key as its **first step** and keeps it for the session only.

- **SKILL.md:** added **Step 0 — Provide the API key** (before Phase 0): ask via `AskUserQuestion`,
  write it to `${TMPDIR:-/tmp}/commotion-mcp/session.env` with `umask 077` (never printed), then
  smoke-test with `GET /aimodel`. Removed the old `.env`-sourcing setup line and the now-redundant
  prerequisites bullet.
- **`scripts/commotion_api.sh`:** auto-loads `COMMOTION_ENV_FILE` (default the session file above)
  when `KONG_API_KEY` isn't already in the environment, so the key set once in Step 0 reaches every
  call (each Bash invocation is a fresh shell). An exported var / local `.env` still takes precedence;
  the not-set error now points at Step 0.
- **`.env.example`:** documented that `.env` is now optional (Step 0 is the default path).
- Security: the key lives only in the session temp file (mode 600) + conversation context — never
  committed, never embedded in the bundle. Bumped `plugin.json` / `marketplace.json` to 0.3.6.

## 2026-06-28 — 0.3.5 — Single-agent prompts CAN render (delete-default-then-POST)

Found + verified live: a `SINGLE_AGENT` worker *can* have a POST-created (UI-rendering, editable)
prompt after all — you just delete the auto-provisioned default first.

- **Recipe:** `DELETE /aiagent/{defaultId}?version=0` (the `version` query param is **required**) →
  agent count 0 → `POST /aiagent` the real agent with `instructions` + `aiAgentEnabled:true`. Its
  prompt renders + is editable. (A direct POST while the default exists is rejected with
  `"Single Agent setup allows only one agent"`.)
- **Setup type is now a pure use-case decision** (single vs multi based on whether the work splits),
  no longer forced toward MULTI_AGENT for prompt visibility. In BOTH setups you POST the
  prompt-bearing agent; the only difference is freeing the slot — single: delete the default; multi:
  disable the default + POST each specialist.
- Updated SKILL.md (Phases 2/6), `agents-and-orchestration.md` (+ `DELETE /aiagent/{id}?version=N`
  in the endpoint tables and `api-and-auth.md`), and the saved memory.

## 2026-06-28 — 0.3.4 — Skill refinements from user review

- **Deploy is always user-gated** — added an explicit `AskUserQuestion` confirmation requirement to
  the intro and Phase 10; never deploy live without a clear "yes".
- **Setup type is inferred, not defaulted** — Phase 2 now tells the model to read the use case and
  prefer `MULTI_AGENT` when the work splits into specialised responsibilities (the user won't say
  "multi-agent"); ask when unsure. Worker reframed as the **orchestrator** of its agents.
- **Single-agent POST is blocked (verified live)** — `POST /aiagent` on a `SINGLE_AGENT` worker →
  `400 "Single Agent setup allows only one agent"`. So a UI-visible/editable prompt requires
  `MULTI_AGENT` + a POST-created agent (a single flow = one specialist + thin orchestrator). Updated
  Phases 2/6 and the agents reference.
- **Models + fallback for voice corrected** — a voice worker's LLM **and its fallback** live in the
  **Voice Settings** block (`workerVoiceSettingsRequest.workerLLMConfigurationRequest`), shown in the
  UI as *Voice Settings → LLM Settings → Fallback Provider/Model* — not `workerAdvancedSettingsRequest`
  (which voice rejects). Fixed `control-and-reliability.md` and `aiworker-lifecycle.md` (previously
  wrongly said "fallback is chat-only").
- **Guardrails are designed from the use case** — Phase 3 and the reference now derive guardrails per
  domain (e.g. banking → PII + card/account masking + competitor forbidden words + toxicity +
  no-advice custom check), not a fixed template.
- **Tools phase reframed** — Phase 8 now says to actively decide what belongs in tools so the prompt
  doesn't do the heavy lifting (ask when unsure), and restates that **every API must be a registered
  tool** (naming an API in the prompt makes the model fabricate `api_call` and loop).
- **Anti-repetition refined** — don't re-ask for info **already given**; re-asking once for info that
  wasn't actually provided (or was unclear) is fine.

## 2026-06-26 — 0.3.3 — UI rendering, tools, language & repetition (HDFC ERGO live test)

Learnings from the user testing the HDFC worker in the Commotion UI:

- **POST-create the prompt-bearing agent, or it won't render/edit in the UI (verified by a
  controlled A/B).** An agent's prompt only shows in the UI prompt editor if the agent was created
  via `POST /aiagent` with `instructions` in the create body. PUT-updating the auto-provisioned
  default agent runs at runtime but leaves the editor blank (the editor doc is set at *create*).
  → A `SINGLE_AGENT` worker (only the un-POST-able default) can't have a UI-editable prompt via API;
  build `MULTI_AGENT` with a thin orchestrator + the real agent **POST-created**. Updated SKILL.md
  (Phases 2/5/6) and `agents-and-orchestration.md`. This also corrects/clears up the long
  "empty prompt editor" investigation (it was never the field, content, size, or trigger cruft).
- **Register APIs as real tools — naming an API in the prompt makes the model fabricate `api_call`.**
  Verified live: with no registered tool, the agent invented `api_call({"api_name":"API 001"...})`,
  got `function 'api_call' is not registered`, and **looped re-asking**. Fix: register each API as a
  custom tool (`POST /ai-worker-tool/custom-tool`) and reference it by action name (`[tool:rmn-check-228]`).
- **Multilingual voice: don't switch language on English digits.** Added a prompt rule (and skill
  guidance): stay in the caller's language; treat numbers/policy-numbers/amounts read in English as
  data; never trigger the `Switch Language` action because of them.
- **Anti-repetition:** never re-ask for given info; call each tool once; on tool error take the
  failure path once — don't loop.

## 2026-06-25 — 0.3.2 — Corrections from live-testing the agent (HDFC ERGO)

Ran the HDFC worker via `POST /aiworker/run` to evaluate real behaviour (not just CRUD). Findings &
fixes:

- **Agent prompt drives runtime, may not show in UI editor.** API-set `instructions` are what the
  agent actually runs on (the live run followed the prompt) even though the UI rich prompt editor can
  render empty. An empty editor box ≠ no prompt. Documented in SKILL.md Phase 6 + agents reference.
- **Corrected a wrong 0.3.1 note:** the agent type DOES stick — the request field `agentType` is
  echoed back as **`aiAgentType`** (the `agentType` key is request-only and reads `null`). Fixed
  SKILL.md, `agents-and-orchestration.md`, and the saved memory.
- **Hallucination from un-wired tools (the key finding).** A prompt that *names* APIs ("call API
  001/002…") with no tools wired makes the agent **fabricate** results — it declared a phone number
  "not registered" with nothing backing it. Added an explicit anti-hallucination rule to the HDFC
  prompt (redeployed v1) and a prompt-authoring rule to SKILL.md Phase 3 + agents reference: never
  assert a backend fact without a tool result; wire the APIs as tools to make it functional.
- **Added Phase 12 (test the agent)** to SKILL.md and documented `POST /aiworker/run` in
  `api-and-auth.md` — the skill now ends by exercising branches/failure paths and checking for
  hallucination, not at "deployed".

## 2026-06-25 — 0.3.1 — Corrections from the first live build (HDFC ERGO Health Renewal)

Dogfooded the skill end-to-end by building + deploying a real production voice worker — the HDFC
ERGO Health Renewal flow (`6a3cfb5a5d29f47d6e6b08c7`, dev3 `demo_workspace`): SINGLE_AGENT, voice
(en+hi, S2S `commotion-laya-v1-5`), toxicity + Commotion-PII guardrails, full 14-section renewal
flow in the agent instructions. Fixes from what the live run surfaced:

- **Scripts-dir resolution:** the `${CLAUDE_PLUGIN_ROOT:?…}` line hard-failed when running from a
  clone (the var is only set for an installed plugin). Replaced with a `:-` fallback + guidance.
- **Prompt placement (correctness):** for SINGLE_AGENT the detailed system prompt / flow logic goes
  in the **agent's `instructions`**, not `workerLevelPrompt` (which is a short role line) — verified
  against a live worker. Fixed SKILL.md Phase 3/6 and `agents-and-orchestration.md`.
- **Voice default agent:** documented that a voice worker's default agent is **"Voice Agent"** with
  `agentType: null`, and that `agentType: "VOICE_AGENT"` doesn't stick (harmless; `aiAgentEnabled`
  is the deploy gate). Fixed SKILL.md Phase 5/6 and `agents-and-orchestration.md`.
- **Voice config:** corrected the live pipeline set (HALF_CASCADE / SPEECH_TO_SPEECH / COLLOQUIAL —
  no FULL_CASCADE/TRANSCRIPTION_BASED) and added a verified-good en+hi voice block to
  `aiworker-lifecycle.md`.
- Knowledge (P7) and tools (P8) were correctly skipped: the flow is API-driven, and wiring the
  HDFC APIs as custom tools needs their real endpoint specs (the prompt encodes every call-point).

## 2026-06-25 — 0.3.0 — Skills-only: call dev3 directly over HTTP (drop the MCP server)

Restructured the plugin so the skills call the Commotion dev3 backend **directly over HTTP** (via
Kong) instead of through the hosted `commotion-mcp` server. Modeled on Cekura's skills: the skill
carries the endpoints and fetches request schemas live from the OpenAPI spec.

- **Removed** `.mcp.json` and all `mcp__commotion__*` tool calls. The `commotion-mcp` repo is left
  in place but is no longer referenced.
- **Added** `scripts/`: `commotion_api.sh` (authenticated request wrapper — injects the Kong base
  URL + `apikey` + `X-Route-Selector`, keeps the key off the command line, surfaces backend errors),
  `fetch_schema.sh` (fetches `/v3/api-docs/public` once per session, caches it, bundles a named
  schema with its `$defs`), and `bundle_schema.py` (stdlib port of
  `commotion-mcp/server/utils/openapi.py:bundle_schema` for byte-equivalent schemas).
- **Rewrote** `commotion-create-worker/SKILL.md` into explicit phases (P0 ground → P1 interview →
  P2 setup type → P3 draft → P4 approve → P5 create → P6 enable agents → P7 knowledge (optional) →
  P8 tools (optional) → P9 readiness → P10 deploy → P11 confirm), each with the exact endpoint + the
  reference to read; `allowed-tools` is now `Bash, Read, AskUserQuestion`.
- **Ported** all reference files MCP→HTTP (endpoint tables added; every gotcha/error-string/field
  path preserved) and added `references/api-and-auth.md` — the endpoint map, header contract, error
  semantics, and `fetch_schema.sh` schema-name list.
- **Config:** `.env` now carries `KONG_API_KEY` (secret) + `KONG_BACKEND_URL` /
  `KONG_API_KEY_HEADER` / `KONG_ROUTE_SELECTOR` (non-secret defaults); added `.env.example`.
  Bumped `plugin.json` / `marketplace.json` to 0.3.0 and updated descriptions.
- **Verified live** against dev3 through the new transport: `GET /aimodel`, `GET /aiworker/metadata`,
  and `fetch_schema.sh AiWorkerRequest` (28 `$defs`, includes `guardrailConfigRequest`), with the
  spec cached and reused across schema names.
