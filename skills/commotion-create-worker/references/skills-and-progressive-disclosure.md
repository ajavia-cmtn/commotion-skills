# Skills — named instruction blocks the agent loads on demand

A **skill** is a named, described block of instructions stored beside the worker instead of inside the
agent prompt. The agent does not receive its contents up front: it receives a one-line advert, and
pulls the body in only when a turn actually needs it. Reached through the two Commotion MCP tools
(`commotion_request` / `commotion_schema` — see `api-and-auth.md`). Field *shapes* come from
`commotion_schema` `{ "schema_name": "AiWorkerSkillRequest" }`; this file is the *behaviour* the schema
doesn't tell you. Everything here was verified live against dev3 on 2026-08-10.

## Endpoints

| Method | Path | Purpose | Schema |
|--------|------|---------|--------|
| GET | `/ai-worker-skill?aiWorkerId=&version=` | list a version's skills | — |
| POST | `/ai-worker-skill` | create a skill on a draft | `AiWorkerSkillRequest` |
| PUT | `/ai-worker-skill/{aiWorkerSkillId}` | full update (keeps the id) | `AiWorkerSkillRequest` |

`AiWorkerSkillRequest` — **all five fields are required**: `aiWorkerId`, `version`, `name`,
`description`, `instructions`. The id field on the response is **`aiWorkerSkillId`**, not `id` (same
trap as `pronunciationDictId`).

```jsonc
{
  "aiWorkerId": "6a797e16c446f4e22b44da44",
  "version": 0,
  "name": "CancellationPolicySkill",          // unique within the worker version; the read_skill key
  "description": "The clinic's cancellation and no-show policy: notice windows, fees, and waiver
                  rules. Use whenever the caller asks to cancel, reschedule, or asks about charges
                  for missing an appointment.",   // ← the routing signal. See below.
  "instructions": "# Cancellation & No-Show Policy\n\n- Free cancellation: more than 48 hours…"
}
```

## How a skill actually reaches the model — progressive disclosure

This is the whole point of the feature, and it is invisible from the schema. At runtime the agent's
system prompt carries only a **manifest** — every attached skill's `name` and `description`, and
nothing else — plus an auto-registered built-in tool **`read_skill(name)`**:

```xml
<available_skills>
  <usage>When a request matches a skill below, call read_skill(name) to load its full instructions
  before acting. Do not guess a skill's contents. Only call read_skill again for a skill if you no
  longer have its full instructions from a previous call.</usage>
  <skill name="CancellationPolicySkill" description="The clinic's cancellation and no-show policy…" />
  <skill name="ReschedulingFlowSkill"   description="Step-by-step procedure for moving an appointment…" />
</available_skills>
```

**Verified live 2026-08-10.** Asked *"I need to cancel my appointment tomorrow, will I be charged?"*,
the agent called `read_skill("CancellationPolicySkill")`, received the body as a tool result, and only
then answered with "48 hours" / "40 percent" / "first offence waived" — figures that exist nowhere but
inside that skill. The token trace shows the saving: **870 input tokens** on the manifest turn, **1053**
on the answering turn. The body costs context **only on turns that use it**.

So: a skill is not a way to write a longer prompt. It is a way to keep a long prompt *out* of the
context window until the moment it matters.

## When to reach for a skill

| Use | Why |
|-----|-----|
| **The prompt is too long** — the agent is ignoring instructions buried in a wall of text | Split each self-contained topic into a skill. The prompt shrinks to routing; each body loads on demand. |
| **Per-topic policy blocks** — cancellation rules, eligibility criteria, fee tables | Only the topic in play is ever in context, so the model cannot cross-contaminate one policy with another. |
| **A procedure several agents share** | One skill, referenced from each agent's `aiWorkerSkillIds` — one place to edit. |

Not a skill:

- **Retrieval over documents** → knowledge / RAG (`knowledge-and-rag.md`). A skill is a fixed block you
  author; knowledge is a corpus the platform chunks and searches.
- **Anything with a side effect or a live lookup** → a tool (`tools-and-capabilities.md`). `read_skill`
  only returns text you already wrote.

## Binding a skill to an agent

**`aiWorkerSkillIds` is the binding. `[skill:<name>]` is only a hint.**

```jsonc
// in the AiAgentRequest body — PUT /aiagent/{id}
"aiWorkerSkillIds": ["6a797e38c446f4e22b44da58", "6a797e52dc3b9ac71250c6c7"]
```

**Attaching a skill is two steps, and you always do both:** bind the id in `aiWorkerSkillIds`, *and*
reference the skill from the agent's `instructions` with `[skill:<name>]`, saying when to use it.

The manifest is built from `aiWorkerSkillIds`, so the id array is what makes the skill reachable at
all. The `[skill:<name>]` token is a routing instruction to the LLM in prose — it is **never
expanded**, and stays literal in the resolved `finalInstructions` (verified in Call Analyzer). Write
it exactly as you would write "always use the [tool:…] for this": it puts the skill in the agent's own
procedure, at the point in the flow where it belongs, instead of leaving the match to description
similarity alone. That is what makes the routing legible to the next person reading the prompt, and
reviewable in the readiness gate.

The mirror-image failure of a dangling `[tool:…]`: a prompt that says `[skill:Foo]` while `Foo` is not
in `aiWorkerSkillIds` is a **dangling skill reference**. There is no error; the model simply has no such
skill to read and will usually answer from general knowledge instead. Check it in Call Analyzer —
`config.skills[]` carries `isReferenced` and `referencedBy` (see `commotion-debug/references/call-analyzer-api.md`).

`aiWorkerSkillIds` also exists on `AiWorkerRequest`. Bind at the **agent** level — that is what drives
the manifest in a single-agent worker, and in a multi-agent worker it is what lets each specialist
advertise only its own skills.

## What a skill body may reference — and how each kind resolves

A skill's `instructions` can carry the same mention tokens as an agent prompt, and **they work from
inside the skill**: a token that only ever appears in a skill body still resolves at runtime, once the
skill is loaded. All three verified live 2026-08-10 on one worker.

| In a skill body | Resolves via | Evidence |
|---|---|---|
| `[skill:<name>]` | a further `read_skill(name)` call | `ReschedulingFlowSkill` → `read_skill("CancellationPolicySkill")` |
| `[tool:<action name>]` | the registered action, called directly by name | `read_skill(...)` → `check-slot-availability-2136` fired |
| `[knowledge:<name>\|id:<id>]` | `search_knowledge_base(query, filters:[{key:"id", value:<id>}])` | `DirectionsAndAccessSkill` → the KB search, filtered to that id |

The pattern is uniform: **every reference type is plain literal text in the prompt, and the runtime
turns each into a tool call.** Nothing is string-substituted. That is why the UI can render all of
them as chips while the model still sees the raw token.

**⚠ The `|id:` in a knowledge token is not decoration — it becomes the retrieval filter.** The
observed call was `search_knowledge_base` with `filters: [{"key": "id", "value": "<the id from the
token>"}]`. This is the reason knowledge tokens carry an id and tool/skill tokens do not: the id
scopes the search to that one document. Get it wrong and the search silently returns nothing.

⚠ **Tool action names are re-minted on a version bump, and the platform rewrites the tokens for you.**
The same tool was `check-slot-availability-2135` at v1 and `check-slot-availability-2136` at v2, and
the `[tool:…]` token *inside the skill body* was rewritten to match when the draft was forked
(verified live). Two consequences: don't hand-copy an action name across versions, and always re-read
`GET /ai-worker-tool?aiWorkerId=…&version=<the version you are editing>` before writing a new token.

**⚠ A rule inside a skill body is not a stronger rule.** In testing, `ReschedulingFlowSkill` said
"Never invent slots. Offer at most three of whatever the tool returns." The agent called the tool,
got back `{"status":"ok"}` with no slots in it — and then invented three plausible ones anyway. Moving
an instruction into a skill does not harden it; the grounding rule ("never assert a backend fact a
tool didn't return") still belongs in the **agent prompt**, where it applies to every turn, loaded
skill or not. See `agents-and-orchestration.md`.

## Writing the `description` — this is the part that decides everything

The `description` is the **only** thing the model sees when deciding whether to load a skill. Treat it
exactly like a tool description, not like a title:

- Say **when to use it**, in the caller's terms: *"Use whenever the caller asks to cancel, reschedule,
  or asks about charges for missing an appointment."*
- Name the concrete nouns a caller would actually say (fees, no-show, waiver), not internal jargon.
- Where two skills are adjacent, say what each is **not** for, or they will both load — or neither will.

A vague description ("Cancellation info") is the single most common way a skill silently never fires.
When a spot-check shows the agent inventing an answer that a skill already covers, **fix the
description before you touch the body** — the body was never read.

## The edges

- **Skills compose, and the chaining works at runtime (verified live 2026-08-10).** A skill's
  `instructions` may itself contain `[skill:…]`, `[tool:…]` and `[var:…]`. Asked to reschedule, the
  agent called `read_skill("ReschedulingFlowSkill")`, whose step 3 says to consult
  `[skill:CancellationPolicySkill]` — and it then called `read_skill("CancellationPolicySkill")`
  before answering. The token ladder across that one turn: **881 → 1078 → 1261** input tokens, each
  step paying only for the skill it just pulled in.
- **A skill body is re-read only when it falls out of context.** The manifest's `<usage>` tells the
  model not to call `read_skill` again while it still holds the instructions, so a long conversation
  does not re-pay for the same skill every turn.
- **`name` is unique per worker version.** A repeat → `400 "Skill creation failed: Skill with name
  'X' already exists."` The name is the `read_skill` key, so keep it short and unambiguous.
- **⚠ Skill ids are not validated for ownership.** Putting a skill id belonging to a *different* worker
  into `aiWorkerSkillIds` returns **200** and is stored silently. Nothing warns you and the skill will
  not resolve. Only ever bind ids you got from
  `GET /ai-worker-skill?aiWorkerId=<this worker>&version=<this version>`.
- **Ids survive a version bump (verified live 2026-08-10).** `POST /aiworker/{id}/draft?version=0`
  carried both skills into v1 with their `aiWorkerSkillId` unchanged (only `createdDate` was
  restamped) — the same rule that holds for agent ids (`aiworker-lifecycle.md`). So an agent's
  `aiWorkerSkillIds` keeps resolving across revert → edit → redeploy; you do not need to re-bind.
- **Skills are `(worker, version)`-scoped**, created on a draft like tools and state variables. Always
  pass the version you are editing.
- **`PUT` is a full replace** — resend `name`, `description` *and* `instructions`, or you will blank
  what you left out.
