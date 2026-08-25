# Decisions

Settled. Don't relitigate without a reason.

## Product
- The AI does intake only. A licensed human finalizes every enrollment.
- The AI never names a carrier or plan, quotes a price, or judges eligibility.
- Marketing never talks down the AI. The closing emphasis is always that a real
  licensed human seals the deal.
- Callers are **inbound**, on hold. Not outbound cold calls.

## Script
- Branching, not flat. Dual-eligible, Medicare-only, and Medicaid-only callers
  each get a different follow-up.
- Doctors, medications, and pharmacy are three separate questions.
- Phone number is captured early, before ZIP.
- Consent decline does **not** end the call. The intake continues; the agent just
  stops naming plan types.
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
- Vapi over Retell, because coding help makes the extra flexibility worth it.
- n8n (not Zapier) for post-call automation.
- Budget target: usage-based, roughly $2k/month ceiling, no big upfront cost.
- CMS-required call recordings and retention are held at **Ritter** — not stored
  in house.

## Transfer failure handling
*Decided 2026-08-25.*

- Transfer mode is **warm transfer** (`warm-transfer-experimental`), not blind.
  Vapi dials the agent and holds the caller with ringtone/hold audio. The calls
  connect only if someone actually answers.
- Destination is a **Twilio ring group that rolls to voicemail**, so there's
  somewhere to land.
- A **fallbackPlan message** plays if the agent doesn't answer or voicemail is
  detected. Exact wording lives in `config/vapi-settings.md`.

**Constraint that drove this:** Vapi cannot resume the assistant conversation
after a failed transfer. An earlier plan — have the AI come back and run the
callback path — is not possible. The fallback message gets one sentence, then
the call ends. So that sentence has to carry the whole explanation.

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
