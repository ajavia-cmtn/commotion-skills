# Large prompts — writing and editing instructions you cannot retype

The failure this file exists to prevent: a worker whose 60 000-character prompt was **quietly
replaced by a shorter paraphrase**. Nothing errors. The `PUT` returns `200`. The UI shows a prompt.
It is just not the prompt the customer wrote. Reported by the team on 2026-08-31 and reproduced below.

## Where the limit actually is (measured, dev3, 2026-08-31)

**Not in the backend.** A single `PUT /aiagent/{id}` with a **128 032-character** `instructions`
field returned `200` and echoed back **128 032 characters** — byte-identical, 800 numbered rules
intact. There is no practical size cap and no truncation on the platform side.

The limit is **you**. The prompt has to travel as a JSON string argument inside one tool call, so
every character is a token you generate. 60 000 characters is roughly 15–20 k output tokens emitted
in a single uninterrupted span, and long verbatim reproduction is the thing models are worst at:
sections get compressed, near-duplicate rules get merged, the tail gets summarised, `\n\n` becomes
`\n`. **`PUT` is a full replace**, so this applies to *edits* too — changing one line in a 60 k prompt
means re-emitting all 60 k, and the other 59 950 characters are where the damage happens.

So the rule is not "be careful". It is: **stop routing large prompt text through your own output.**

**Two independent reports, same day.** A second team hit this on tcuat with a **74 000-character**
prompt and reached the same diagnosis unaided: *"There's no PATCH (full body only), so applying it
means re-sending the whole 74k-char prompt."* Correct — `/aiagent` has `POST`, `PUT` and `DELETE` and
no `PATCH`. One correction to that write-up, because it changes what you should worry about: `version`
on `AiAgentRequest` is **not** optimistic locking on the agent record. It is *"configuration version of
the parent worker — must match the worker's current version"*, i.e. draft-version pinning. The risk is
writing to the wrong worker version, not losing a race with a concurrent editor.

**When the write gets blocked, do not route around it.** That same session responded to a permission
denial by generating an `apply.py` and asking the user to `export COMMOTION_UAT_TOKEN=… && python
apply.py --write` against a **guessed** base URL. Don't. The staging half was exactly right — patched
file, original kept for revert, byte-identity check after write — so keep that and change only the
last step: ask the user to approve the `commotion_request` write, or point them at the
`permissions.allow` rule in the README. See "When a call is refused by the client, not by the backend"
in `api-and-auth.md`.

## The size gate — decide before you write anything

Measure first. If the text is in a file, `wc -c`; otherwise estimate from the source you were given.

| Size | What to do |
|------|------------|
| **< 8 000 chars** | Write it inline. Normal Phase-6 `PUT /aiagent/{id}`. Nothing special. |
| **8 000 – 20 000** | File-as-source-of-truth + the read-back gate below. |
| **> 20 000** | The same, **and** raise decomposition with the user before you write (next section). |
| **any size, user says "keep it exactly"** | Verbatim path + read-back gate. Never paraphrase. |

## Above ~20 k: say the quiet part out loud

A 60 000-character monolithic prompt is not just hard to write — it is a worse worker. Every token is
in the context of every turn, on every call, forever: it costs latency and money per turn, and it
dilutes the instructions that actually matter for the turn in hand.

The platform already has the answer — **Worker Skills** (`references/skills-and-progressive-disclosure.md`):
a short core prompt plus named skill bodies the agent loads **on demand**. A 60 k monolith usually
decomposes into a ~3 k core plus 8–15 skills of 2–5 k each. Each piece is small enough to write
without drift, each is independently editable, and the runtime only pays for what a turn actually uses.

**Offer it, don't impose it.** Use `AskUserQuestion`: keep it as one prompt (works, but heavier at
runtime and every future edit re-writes the whole thing) or split it into skills (better runtime,
more moving parts, and the split itself is a judgement call they should see). If they want it in one
piece, do it in one piece — via the verbatim path below.

## The file is the source of truth

Never let a large prompt exist **only** in the conversation. From the moment you have it:

1. `Write` it to a working file — `./worker-prompts/<worker-name>.<agent-name>.md`.
2. **Every subsequent change is an `Edit` on that file**, never a rewrite from memory. `Edit` replaces
   an exact string and leaves every other byte untouched — which is precisely the guarantee that
   retyping the prompt does not give you.
3. `wc -c` the file before and after, and say the delta out loud. A one-line fix that changes the
   file size by 4 000 characters means you rewrote something you did not mean to.
4. When you send it, you are transcribing **one file you just read**, not reconstructing from memory
   across a long conversation. That is a much easier task and a much better failure profile.

This also gives the user something to review and keep: the prompt is a source artifact, not a
transcript scroll-back.

**Editing an existing large prompt**: `GET /aiagent/{id}` → `Write` the returned `instructions`
straight to the file → `Edit` the file → send the file back. Do not go `GET` → *think* → `PUT`.

## Structure the prompt so drift is detectable

Author large prompts with **numbered section anchors** — the whole verification strategy depends on it:

```markdown
## S01 — Identity and tone
...
## S02 — Greeting
...
## S17 — Escalation to a human
```

Two lines of cost, and it turns "did anything get lost?" — unanswerable on a 60 k blob — into a
countable check. Keep the anchors in the deployed prompt; they cost ~20 tokens each and they make
every future edit and every future audit tractable.

## The read-back gate — mandatory above 8 k

**A `200` is not evidence.** The backend echoes what it received, and what it received is what you
emitted, drift included. So after the `PUT`:

1. `commotion_request { "method": "GET", "path": "/aiagent/<id>" }`.
2. Check, in this order:
   - **Every anchor is present, once, in order** — `S01 … Snn`, none missing, none duplicated.
   - **First line and last line** match the file exactly.
   - **Length is in range.** You know the file's `wc -c`; the returned `instructions` should be within
     ~1 % of it. A 10 % shortfall is a dropped section, not rounding.
3. **On any mismatch: stop.** Do not deploy, do not "fix it up". Re-send the whole field from the file
   and gate again. If the second attempt also drifts, **tell the user plainly** which sections were
   lost and hand them the file — a wrong prompt shipped silently is far worse than an honest failure.

Never report a large-prompt write as done without stating that the gate passed and what you checked.

## Hard rules

- **Never paraphrase, compress, "clean up" or reflow a prompt the user gave you.** Not to save tokens,
  not to fix its formatting, not because a section looks redundant. If you believe it should change,
  propose the change and let them decide.
- **Never write a prompt you could not reproduce in full.** If you cannot, say so and stop. There is
  no acceptable partial write — the field is a full replace, so a partial write is a deletion.
- **Never re-derive a prompt from a summary of itself.** Once it is in the file, the file wins over
  anything in your context, including your own earlier reasoning about it.
- **Never send a large `PUT` with fields you did not restate.** `PUT /aiagent/{id}` replaces the whole
  record: `name`, `description`, `agentType`, `aiAgentEnabled`, `modelConfigurationRequestList`,
  `aiAgentTriggerInputList` all have to be there, or you blank them while fixing the prompt.

## Known limitation — and what would actually fix it

Everything above is damage control. The structural fix is transport-side, in the Commotion MCP
server, and is **not yet available**: a `commotion_text_patch`-style tool that takes
`{ path, field, edits: [{ old_string, new_string }] }`, performs the read-modify-write **server-side**,
and returns the resulting length and checksum. Then a one-line change to a 60 k prompt costs the two
lines that changed instead of 60 000 characters, and verification is a hash comparison rather than an
eyeball. Until that exists, the file + anchors + read-back gate is the best available guarantee — and
you should tell users asking for repeated edits to a very large prompt that each edit currently
rewrites the whole field.
