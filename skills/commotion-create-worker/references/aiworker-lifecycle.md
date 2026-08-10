# dev3 aiworker lifecycle — draft/live, versions, and the sharp edges

Operational behavior of the `/aiworker` backend, reached through the two Commotion MCP tools
(`commotion_request` / `commotion_schema` — see `api-and-auth.md`). Read this when a create / update /
deploy call behaves unexpectedly. Field *shapes* come from `commotion_schema` `{ "schema_name":
"AiWorkerRequest" }`; this file is the *behavior* the schema doesn't tell you. Agent specifics live in
`agents-and-orchestration.md`.

## Endpoints

| Method | Path | Purpose |
|--------|------|---------|
| POST | `/aiworker` | create → DRAFT v0 |
| PUT | `/aiworker/{id}` | update the draft (full PUT; body needs `version`) |
| POST | `/aiworker/{id}/deploy?version=N` | deploy version N → LIVE |
| POST | `/aiworker/{id}/draft?version=N` | revert a live worker to a new editable draft |
| GET | `/aiworker` | list (live + draft) |
| GET | `/aiworker/{id}` | retrieve **live** (`?version=N` for a specific version) |
| GET | `/aiworker/{id}/versions` | version history |
| GET | `/ai-worker-skill?aiWorkerId=&version=` · POST/PUT `/ai-worker-skill[/{aiWorkerSkillId}]` | a version's skills — see `skills-and-progressive-disclosure.md` |
| GET | `/aiworker/metadata` · `/aimodel` | valid values / supported **language** models |
| GET | `/aiworkervoice?providerId=&modelId=&languageId=&accentId=` | **the TTS voice catalogue** (`voiceId`, `voiceName`, sample audio) — one word, no separator |

## State machine

```
POST /aiworker                        -> DRAFT, version 0     (auto-provisions a default agent, DISABLED)
PUT  /aiagent/{id} | POST /aiagent    -> edit agents          (only while the worker is a DRAFT)
PUT  /aiworker/{id}  (body w/ version)-> edit the draft       (full PUT; needs `version`)
POST /aiworker/{id}/deploy?version=0  -> LIVE, version 0
```

Editing something that is already **LIVE**:

```
POST /aiworker/{id}/draft?version=0
        -> creates a NEW editable DRAFT version (e.g. v1) alongside the still-serving LIVE v0
edit the draft (worker config and/or its agents) at that new version
POST /aiworker/{id}/deploy?version=1  -> the draft becomes LIVE; the old version flips to PAUSED
```

`GET /aiworker/{id}/versions` shows the history, one entry per version with its `status` —
**`LIVE` / `DRAFT` / `PAUSED`**. Only **one draft can exist at a time** per worker.

**The superseded version becomes `PAUSED`, not deleted (verified live 2026-08-07).** After deploying
v1 on two workers, `GET /aiworker/{id}/versions` returned `[{version:0, status:"PAUSED"},
{version:1, status:"LIVE"}]` on both. So version history accumulates and old versions stay readable
(`GET /aiworker/{id}?version=0`) — don't read a `PAUSED` entry as a failed or rolled-back deploy.

**⚠ The whole edit cycle preserves agent ids (verified live 2026-08-07).** Reverting a live worker to
a draft carries each agent into the new version with its **`aiAgentId` unchanged** — confirmed across
five agents of four different types on two workers (only `createdDate` was restamped). Combined with
`PUT` as the prompt path, **no routine operation churns an agent id**: revert → `PUT` → redeploy is
id-stable end to end, so scenarios, ACW references and anything else pinned to `aiAgentId` survive a
version bump. (Delete + re-POST still mints a new id — that is the one thing that breaks the chain.)

**Skill ids ride along too (verified live 2026-08-10).** The same revert carried both of a worker's
skills into v1 with their `aiWorkerSkillId` unchanged, so an agent's `aiWorkerSkillIds` keeps resolving
across the bump — no re-binding needed. Same rule, same evidence: only `createdDate` was restamped.

## The edges

- **A new worker is DRAFT v0 and comes with a default agent, disabled.** It is not usable until that
  agent is enabled (`aiAgentEnabled: true`). See `agents-and-orchestration.md`.
- **`agentSetupType` is required on create** (`SINGLE_AGENT` / `MULTI_AGENT` / `WORKFLOW`); blank →
  `400 "Agent setup type cannot be blank"`. It **can be changed later via `PUT /aiworker/{id}`, but
  only while the worker is a draft** (verified: a SINGLE_AGENT draft was switched to MULTI_AGENT,
  then redeployed).
- **Update is a full PUT and needs `version`.** `PUT /aiworker/{id}` requires the current `version` in
  the body (`0` for a fresh worker, or the draft's version when editing a live worker's draft) — else
  `400 "version is required for update"`. It replaces the record, so resend the top-level fields you
  want to keep (`name`, `workerGoal`, `workerLevelPrompt`, `agentSetupType`, `voiceEnabled`, the
  voice block) or they reset.
- **Deploy targets a version.** A fresh worker's first deploy is `version=0`. When you revert a live
  worker to a draft, the draft gets a **new** version number (e.g. 1); deploy *that* version to go
  live. Workers can be LIVE at version 0.
- **`GET /aiworker/{id}` is LIVE-only.** A draft-only worker returns `400 "Live Worker not found"`.
  Read drafts from `GET /aiworker` (list), or fetch a specific version with `GET /aiworker/{id}?version=N`.

## Voice + language config

Languages and the TTS voice live under the voice settings block of the `AiWorkerRequest`:

```
workerVoiceSettingsRequest:
  voiceAgentPipelineType: "SPEECH_TO_SPEECH"   # live options (GET /aiworker/metadata voiceConfig):
                                               #   HALF_CASCADE / SPEECH_TO_SPEECH (default) / COLLOQUIAL
  workerVoiceConfiguration:
    allowedLanguages: ["en", "hi"]             # every language the worker may speak
    language: "English-Indian"
    defaultLanguage: "en"
    model:    "<tts model>"                    # REQUIRED — e.g. commotion-laya-v1-5
    provider: "<tts provider>"                 # REQUIRED — e.g. commotion-tts
    voiceId:  "<voice id>"                      # REQUIRED — a UUID
  workerTranscriptConfiguration: { provider, model, temperature, prompt }   # optional; defaults stand
  workerLLMConfigurationRequest: { provider, model, temperature, + fallback fields }  # the voice worker's LLM lives HERE
```

**For a voice worker, the LLM (brain) AND its fallback model are configured in this voice block**
(`workerLLMConfigurationRequest`) — this is the UI's *Voice Settings → LLM Settings → Provider /
Model / Fallback Provider / Fallback Model*. Do NOT use `workerAdvancedSettingsRequest` on a voice
worker (it's rejected). Inspect the exact fallback field names with `commotion_schema` `{ "schema_name":
"AiWorkerRequest" }` (`WorkerVoiceSettingsRequest`). See `references/control-and-reliability.md`
("Models + fallback").

**Verified-good en+hi block (mirror this — created + deployed live, S2S):** `provider:
"commotion-tts"`, `model: "commotion-laya-v1-5"`, `voiceId:
"d6d81480-227c-41cd-af4e-f483262cef0b"` (that UUID is the voice **Poornima** — pick deliberately from
`GET /aiworkervoice`, don't paste it blindly; see "Choosing a voice" below).
`commotion-laya-v1-5` covers `en, hi, kn, mr, ta, te, ml, bn, pa, gu`. Sending **only**
`voiceAgentPipelineType` + `workerVoiceConfiguration` on create is enough — the transcript/LLM
sub-blocks default. The full provider→model→language map lives in
`GET /aiworker/metadata` → `voiceConfig.voicePipelineTypeConfig`.

## The first message — worker-level `greetingInstructions`

A top-level string on `AiWorkerRequest` (agents carry their own; see `agents-and-orchestration.md` for
the field's behaviour and its unenforced no-references rule). It survives deploy and revert-to-draft.
Set it on the worker when the opening line is the same whoever picks up; set it per agent when each
specialist should open differently.

## After Call Work (ACW) — hand the finished call to a second worker

ACW fires a **separate worker** once the call ends, to do post-call analysis, CRM updates and
follow-ups. The convention is a non-voice worker named **`[ACW] <worker name>`** whose single agent
does the wrap-up; you reference that worker and that agent.

```jsonc
workerVoiceSettingsRequest: {
  workerAdvancedVoiceSettingsRequest: {
    acwMetadataRequest: {
      aiWorkerId: "<the ACW worker id>",   // REQUIRED — must be LIVE
      aiAgentId:  "<its agent's id>",      // REQUIRED — must be LIVE
      includeSessionState: true,           // send the voice session state along with the transcript
      transcriptFormat: "MARKDOWN"         // JSON | MARKDOWN
    }
  }
}
```

Build order, and it is not negotiable: **create the ACW worker → configure its agent → deploy it →
only then reference it.** Pointing at a draft fails with `400 "ACW live worker not found with ID:
<id>"` (verified live 2026-08-10).

- **⚠ Use the worker-level block, not the agent one.** `AiAgentRequest.advancedSettingsRequest` also
  advertises an `acwMetadataRequest` (with two extra fields, `acwDisabled` and `voiceInputTypeList` —
  the latter accepts `TRANSCRIPT` | `AUDIO` but only `TRANSCRIPT` is supported today). It is
  **unreachable for a voice agent**, which rejects the whole advanced-settings block — see
  `agents-and-orchestration.md`. Since ACW is a voice feature, worker level is the real path.
- **⚠ Don't send `voiceAgentType` unless you mean it.** Including it in
  `workerAdvancedVoiceSettingsRequest` pulls in telephony validation:
  `400 "Worker advanced voice settings connectionId cannot be blank when voiceAgentType is given."`
  Omit it and the ACW block saves cleanly on a worker that has no DID yet (verified live 2026-08-10).
- **`PUT /aiworker/{id}` is a full replace**, and ACW now lives *inside* the voice block — resend the
  whole `workerVoiceSettingsRequest` (pipeline type, voice config, LLM config) alongside it, or you
  will reset the voice settings while adding ACW.

## Providers and credentials — pick the provider; the credential follows (verified live 2026-08-06)

**The rule, in one line: choose a first-party Commotion provider for TTS, STT and the LLM — and then
there is no credential to choose, because a first-party provider takes its key from the platform
environment.**

### Provider and credential are not two independent choices

This is the distinction that gets inverted most often, so state it explicitly:

- **First-party Commotion provider** → key comes from the platform env. The credential is resolved
  **automatically** and is not a field you set. In the UI the Credential box fills itself in with
  "Commotion"; over the API there is simply nothing to send.
- **Third-party provider (BYOK)** → the *customer* supplied the key, so a `ClientProviderCredential`
  must be bound. The schema is unambiguous about what a credential is for: `credentialId` is
  *"Credential ID for **BYOK (Bring Your Own Key)** support."*

**⚠ Therefore "Commotion" is never a *credential* to attach to somebody else's provider.** A block
reading *Provider: Eleven Labs · Credential: Commotion* is always wrong — it pairs a third-party
provider with a first-party key. If you find yourself selecting a credential at all for STT/TTS/LLM,
you have already picked the wrong provider. Fix the **provider**, not the credential.

### Which blocks can even carry a credential

Only two, and both are BYOK-only by design:

| Block | Schema | Credential field? |
|---|---|---|
| TTS voice | `WorkerVoiceConfigurationRequest` | **none** |
| STT | `WorkerTranscriptConfigurationRequest` | **none** |
| Voice LLM | `WorkerLLMConfigurationRequest` | **none** |
| Worker primary LLM | `WorkerLanguageModelConfigurationRequest` | **none** |
| Agent primary LLM | `ModelConfigurationRequest` | **none** |
| Worker LLM *fallback* | `WorkerFallbackModelConfigurationRequest` | `credentialId` ✅ (BYOK) |
| Agent LLM *fallback* | `FallbackModelConfigurationRequest` | `credentialId` ✅ (BYOK) |
| Simulator / eval-metric LLM | `LLMConfig` | `voiceProviderCredentialId` ✅ (BYOK) |

`AiModelResponse.credentialId` is documented as *"ID of the ClientProviderCredential this model is tied
to. **Null for platform models**"* — null for platform models is the same fact from the other side.
There is **no endpoint that lists or creates those credentials** (`/ai-worker-tool/credentials` is the
SaaS-connector plane — unrelated), so a third-party STT/TTS/primary-LLM choice can only ever be written
**credential-less**, i.e. broken.

### How to tell first-party from third-party — don't guess, read the flag

`GET /aiworker/metadata` labels every voice provider explicitly. Check it before writing any provider:

```
voiceConfig.voicePipelineTypeConfig[].providerIdToProviderDropdownOutputMap
  → { "commotion-tts": { "label": "Commotion",  "isDefault": true,  "isThirdParty": false },
      "eleven-labs":   { "label": "Eleven Labs", "isDefault": false, "isThirdParty": true  }, … }
voiceConfig.voicePipelineTypeConfig[].transcriptProviderList
  → [ { "code": "commotion-llm", "label": "Commotion LLM", "isDefault": true  },
      { "code": "commotion-asr", "label": "Commotion ASR", "isDefault": false },
      { "code": "eleven-labs",   "label": "Eleven Labs",   "isDefault": false }, … ]
```

**`isThirdParty: false` is the only acceptable value for TTS, STT and the primary LLM.** For LLMs the
equivalent tell is `providerCode: "commotion"` / `providerLabel: "Commotion"` with a null
`credentialId`.

> ### ⚠ Always call `/aimodel` with `pageSize` — the bare call hides every Commotion model
>
> **`GET /aimodel` is paginated and defaults to ~10 rows.** Verified live 2026-08-06: the bare call
> returned **10 junk `azure_openai` test rows** (`test_model`, `gpt-go`, `a`, `b`, `d`, `hello`, …) and
> **not one Commotion model** — which reads exactly like "this workspace has no Commotion models, so I
> must use a third-party provider." That inference is wrong, and it is a direct path into the
> credential-less trap above.
>
> **`GET /aimodel?pageSize=200`** returns the real catalogue: `commotion-3.6-35b` (**`isDefault: true`**),
> `commotion-4-31b-it`, `commotion-3.6-27b` — alongside the BYOK `openai`, `anthropic`, `cerebras`,
> `vayu` and `azure_openai` entries. **Never conclude a provider is unavailable from an unpaginated
> list.** If you still can't find a first-party pair, read an existing worker's
> `workerLLMConfigurationResponse` or an agent's default `modelConfigurationResponseList` instead.

### The backend does not stop you — that is the trap

Verified live 2026-08-06 on a real draft: a `PUT /aiworker` writing a third-party STT provider returned
**`200`, no error, no warning**, round-tripped the third-party provider and model, and carried **no
credential field anywhere in the response**. In the UI that renders as *Provider: <third party> ·
Model: filled · Credential: blank or bound to the wrong key* — saved, and unrunnable. Nothing in the
API response tells you this happened; only the `isThirdParty` check beforehand will.

So:
- **Do** use `commotion-tts` (TTS), `commotion-llm` (STT; `commotion-asr` is the other first-party
  option), `commotion` (LLM). These are platform models, need no credential, and are the only
  combination that works end-to-end over the API. Verified-good trio: TTS `commotion-tts` /
  `commotion-laya-v1-5`; STT `commotion-llm` / `commotion-omni`; LLM `commotion` / `commotion-3.6-35b`.
- **Don't** write a third-party provider "to be fixed later", and **never backfill the model while
  leaving the credential unset** — a half-set block is worse than an unset one, because the defaults no
  longer apply and nothing flags it.
- **Don't** reach for a credential to rescue a third-party provider. There is no credential field on
  those blocks, and no endpoint to mint one.
- If the user explicitly asks for ElevenLabs/Cartesia/Sarvam/OpenAI/Anthropic for STT/TTS/LLM: **say it
  must be done in the UI** (the credential can't be bound over the API), leave the block on Commotion
  or unset, and change nothing else. Only a **fallback** LLM can take a BYOK `credentialId`, and only
  when you already have that id — find it with `GET /aimodel?credentialId=<id>`.

⚠ **Simulator personalities are the legitimate BYOK case** — a personality's caller voice carries
`voiceProvider: "eleven-labs"` **with a real `voiceProviderCredentialId`**, and that is correct: it is
the customer's own key, on a block that has a credential field. Don't "fix" it to Commotion.

## Choosing a voice — `GET /aiworkervoice` (the voice catalogue)

**Voices come from `GET /aiworkervoice`, not `/aimodel`.** `/aimodel` returns *language models*; it has
no voices in it. The endpoint is `/aiworkervoice` — one word, no separator, which is why searching for
`/voice` finds nothing and concluding "voices are UI-only" is wrong.

```
GET /aiworkervoice?providerId=commotion-tts&modelId=commotion-laya-v1-5   # filter to the worker's TTS model
    # also accepts: voiceId, languageId, accentId, pageNumber, pageSize
```

Each record gives **`voiceId`** (the UUID for `workerVoiceConfiguration.voiceId`), **`voiceName`**, a
prose `description`, `languageIds`, `accentIds`, `modelIds`, and `languageIdToVoiceUrlMap` — per-language
sample audio URLs you can offer the user to listen to. Filter by `providerId`+`modelId` so every
returned voice is actually valid for the configured TTS model; it works for external providers too.

`commotion-tts` / `commotion-laya-v1-5` currently has **five** voices (verified live), all covering
`en, hi, kn, mr, ta, te, ml, bn, pa, gu`:

| `voiceName` | `voiceId` |
|---|---|
| Komal (female, upbeat) | `1b50c54f-2fd1-4ccf-9c33-ab2ca8ce94a7` |
| Raj (male, authoritative) | `04bb4f44-164f-4e39-bb8e-69d5ba134d65` |
| Poornima (female, warm) | `d6d81480-227c-41cd-af4e-f483262cef0b` |
| Tanya (female, refined) | `cb28ee98-e1b2-4f8d-8dc5-039d11486013` |
| Abhay (male, calm) | `f4af8e02-9469-4332-b9cf-d3e1590363f9` |

**Always query the catalogue rather than reusing a hardcoded id** — this table is a fallback, not the
source of truth, and `d6d81480…` (quoted elsewhere as "verified-good") is simply Poornima. When the user
says "choose a voice" or "use a different voice", list the candidates with their names and descriptions,
let them pick, then set `voiceId`; don't claim the catalogue is unavailable.

Notes:
- The `workerVoiceConfiguration` sub-object **requires** `model` / `provider` / `voiceId`. Confirm the
  valid pipeline/provider/model/language combinations in `GET /aiworker/metadata` (`voiceConfig`),
  **voices in `GET /aiworkervoice`** (see above), or omit the whole voice block on create and let
  backend defaults stand, then add languages with an update.
- ⚠ **`voiceConfig…accentIdToAccentDropdownOutputMap` is language accents, not voices** (`English-Indian`,
  `Hindi`, `Marathi`, …). It feeds `workerVoiceConfiguration.language`; it will never give you a `voiceId`.
- **TTS provider availability depends on the pipeline.** `SPEECH_TO_SPEECH` and `COLLOQUIAL` expose only
  `commotion-tts`; `HALF_CASCADE` also exposes the external providers (which you should not use — see
  above). Read `voiceConfig.voicePipelineTypeConfig[].providerIdToModelIdsMap` for the live map.
- **Request vs response field names differ.** The request uses `…Request` suffixes
  (`workerVoiceSettingsRequest`, `workerLLMConfigurationRequest`); a list/retrieve *response* uses
  `…Response`. Don't copy a response back as a request — map field-by-field against
  `commotion_schema` `{ "schema_name": "AiWorkerRequest" }`, or the mismatched keys are silently dropped.
- Multilingual workers also need a prompt line telling them to mirror the caller's language: the
  voice config lets them *speak* the language; the prompt makes them *choose* to.
- **Don't switch language on English-spoken digits (verified live — bake into the prompt).** Stay in
  the caller's spoken language and continue the WHOLE call in it. Treat a mobile number, policy
  number, OTP, or amount **read out in English digits as normal data** — do **NOT** switch the
  conversation language, and do **NOT** trigger the `Switch Language` built-in action, just because
  numbers are spoken in English. Only switch when the caller actually changes their conversational
  language. Without this rule the agent flips Hindi→English the moment the caller says their number in
  English digits (observed live). See `references/tools-and-capabilities.md` for the
  `switch_language` built-in.
