# Decisions

Settled. Don't relitigate without a reason.

## Product
- The AI does intake only. A licensed human finalizes every enrollment.
- The AI never names a carrier or plan, quotes a price, or judges eligibility.
- Marketing never talks down the AI. The closing emphasis is always that a real
  licensed human seals the deal.
- Callers are **inbound**, on hold. Not outbound cold calls.

## Script
- The licensed agent identified in the recorded SOA is **James Saint-Just**.
  His contact number is **+1 561-247-1443**, spoken digit by digit as
  "five six one, two four seven, one four four three."
- Branching, not flat. Dual-eligible, Medicare-only, and Medicaid-only callers
  each get a different follow-up.
- Doctors, medications, and pharmacy are three separate questions.
- Phone number is captured early, before ZIP.
- SOA decline does **not** end the call, but N7–N13 are skipped. The assistant
  asks only neutral callback timing, summarizes prior contact facts, and
  transfers. The licensed agent must obtain valid scope before product discussion.
- Veteran question included, with a thank-you line. It is a courtesy and an
  eligibility signal, not a qualifying gate — nothing downstream branches on it.
- Summary read-back stays under ~15 seconds. Essentials only.
- **Asking for a human always wins.** *Decided 2026-08-25.* Any explicit request
  to speak to a person transfers immediately, mid-question, with a partial
  record. The intake exists to save the caller time; continuing to ask questions
  after they've asked for a human converts it into an obstacle.
- **Program status has four real branches plus unsure:** dual, Medicare-only,
  Medicaid-only, and neither. "Neither" is the turning-65 shopper and is an
  ordinary caller, not an error case.
- **The branch map is the spec.** `flows/branching-logic.md` is edited first;
  `prompts/system-prompt.md` implements it. The prompt is one rendering of the
  flow — if the project later moves to a Vapi Squad or Workflow, the map
  survives and the prompt gets rewritten.

## Stack
- Cloud-hosted, not self-hosted. Considered a home NAS build and rejected it.
- Twilio over Telnyx/Plivo for the phone layer.
- **Run on the Twilio number, not Vapi's built-in one.** *Decided 2026-08-25.*
  Warm transfer is documented as Twilio-only, and warm transfer is the whole
  handoff design. Steps in `config/vapi-settings.md`.
- **Measure prompt caching before model downgrade.** *Updated 2026-08-25.* Cache
  reads are discounted but cache writes can cost more, so enable it only when
  observed reuse produces a net saving. A cheaper model still requires the full
  regression suite. See `docs/cost-model.md`.
- **A model change is not shippable until the P0 test block passes.** The test
  suite is what makes trying a cheaper model a measurable experiment instead of a
  gamble.
- **Noise gets removed, not tolerated.** *Decided 2026-08-25.* Background noise
  cutting the agent off is fixed with denoising first (`smartDenoisingPlan`,
  which ships off) and only then by raising the interruption threshold. Barge-in
  is never disabled to solve it — talking over a caller is worse than being
  interrupted by a TV.
- **`stopSpeakingPlan.numWords` stays at 2, not 3.** The interruption that
  matters most on this call is a one- or two-word demand for a human. A higher
  threshold makes the agent talk over exactly the caller it most needs to hear.
- Vapi over Retell, because coding help makes the extra flexibility worth it.
- Post-call automation: **reopened 2026-08-25.** n8n was chosen over Zapier, but
  a scheduled agent polling the Vapi API is a viable alternative that drops the
  webhook host entirely — polling latency is irrelevant here, since the caller
  already went to a human. What is *not* viable is "Claude handles it in chat":
  calls end at 7pm on a Saturday and something has to be running. See
  `automation/README.md`.
- Budget target: usage-based, roughly $2k/month ceiling, no big upfront cost.
- Recording and retention responsibility is unresolved until the Ritter
  contract, full-call capture boundary, retrieval procedure, access controls,
  and retention proof are documented.

## Verified against Vapi docs, 2026-08-25

Everything below in this section was checked against Vapi's live documentation
rather than assumed. Three corrections came out of it:

- **Warm transfer is documented as Twilio-only.** We are on a Vapi-native
  number, so the mode below may never have been in effect. This reorders Phase 1.
- **`warm-transfer-experimental` needs `voicemailDetectionType`**, with only
  Google or OpenAI supported as detection providers for that mode. Without it the
  fallback plan cannot fire.
- **Current assistant-based `warm-transfer-experimental` supports operator
  context and a fallback.** The transfer assistant explicitly calls
  `transferSuccessful` after human acceptance and `transferCancel` for
  voicemail, busy, no answer, or rejection.

## Transfer failure handling
*Decided 2026-08-25.*

- Transfer mode is **warm transfer** (`warm-transfer-experimental`), not blind.
  Vapi dials the agent and holds the caller with ringtone/hold audio. The calls
  connect only if someone actually answers.
- Destination is a **Twilio ring group that rolls to voicemail**, so there's
  somewhere to land.
- A **fallbackPlan message** plays if the agent doesn't answer or voicemail is
  detected. With `endCallEnabled: false`, control returns to the intake
  assistant, which runs N16 and offers callback collection without claiming a
  callback is already scheduled.

Blind transfer was rejected because an unanswered blind transfer drops the caller
into silence, which directly undercuts the "you won't have to repeat yourself"
promise. Returning to the hold queue was rejected because being told an agent is
coming and then landing back in the queue is worse than never being told.

## Language
- English, Spanish, Haitian Creole.
- The offer is spoken in each language at the top of the call.
- On switch, the agent commits fully to that language for the rest of the call.
- **Single assistant for now.** A Vapi Squad (greeter → dedicated per-language
  assistants) was researched and is the better long-term architecture, but is
  deliberately deferred. See `docs/roadmap.md`.
