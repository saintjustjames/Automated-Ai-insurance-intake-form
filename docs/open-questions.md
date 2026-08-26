# Open Questions

Answer these before building past them.

## 1. Transfer with no agent available — DECIDED, needs testing
**Resolved 2026-08-25.** Warm transfer with a fallbackPlan message, destination
pointed at a Twilio ring group that rolls to voicemail. See `docs/decisions.md`.

Still open, and it's a real risk:
- **Likely root cause found.** Vapi documents warm transfer as available *only
  with Twilio-based telephony*, and this project is currently running on Vapi's
  built-in number. The reported symptom — "connects immediately instead of
  holding" — is what warm transfer degrading to blind would look like. Import
  the Twilio number and re-test **before** touching any transfer setting.
- `warm-transfer-experimental` also needs `voicemailDetectionType` set, and Vapi
  documents only Google or OpenAI as supported voicemail-detection providers for
  it. Without that, the fallback message never fires and an unanswered transfer
  drops the caller into a voicemail box with no explanation — the exact outcome
  the fallback exists to prevent.
- Current Vapi assistant-based warm transfer can return the caller to the
  original assistant when `fallbackPlan.endCallEnabled` is `false`. Configure
  the transfer assistant to call `transferCancel` on voicemail, busy, no answer,
  or rejection, then route the returned caller through N16.
- The fallback message makes no scheduling promise. N16 may collect a request,
  but the assistant must not claim it is saved or scheduled until the callback
  workflow is deployed and monitored.

## 2. Haitian Creole transcription quality — BLOCKING for Creole
Spanish is safe. Creole is not verified. Run a live test call in Creole and see
whether the transcript holds up. If it doesn't, pull Creole from the greeting and
route those callers straight to a human instead.

Concretely, if it fails: interrupt GI-12 in `flows/branching-logic.md` should
route Creole callers to GI-4 (immediate transfer) rather than attempt an intake
in a language the transcriber can't reliably hear. A garbled Creole medication
list is worse than no medication list.

## 3. Squad vs. single assistant — ANALYZED, deferred to Phase 2
Full analysis in `docs/multi-assistant.md`.

Conclusion: worth doing, but for **reliability and language quality**, not cost.
Splitting fragments the prompt cache, so it competes with caching rather than
stacking with it, and changes none of the fixed per-minute costs. The one cost
argument that survives is per-assistant model selection — cheap model for
mechanical capture, capable model only where judgment lives.

Two findings worth carrying forward:

- **Vapi no longer recommends Workflows for new builds.** Squads with handoffs is
  the current guidance; Vapi reports that models can't reliably hold a node's
  instructions plus all possible next steps. So this is a Squad, not a node graph.
- **Global interrupts and consent flags must be replicated in every assistant.**
  If `plan_naming_suppressed` doesn't survive a handoff, a later assistant names
  plan types to a caller who declined consent — a compliance failure, not a
  cosmetic one. Same for the emergency stop and the immediate-transfer interrupt.

Deferred until English is proven end to end, and introduced with Spanish, where
a language router is one decision at the top of the call rather than five
mid-call state transfers.

## 4. Who works failed-transfer callbacks?
Assign an owner and SLA before enabling callback promises. Until the authenticated
workflow creates and monitors a callback task, N16 may collect a request but must
not say it is saved, scheduled, or guaranteed.

## 5. Transfer mode — RESOLVED IN CURRENT VAPI DOCS, needs deployment
Use assistant-based `warm-transfer-experimental`. Current Vapi documentation
supports an operator summary or transfer-assistant conversation together with a
fallback plan. With `fallbackPlan.endCallEnabled: false`, a failed transfer can
return to the original assistant. The transfer assistant must call
`transferSuccessful` after human acceptance and `transferCancel` for voicemail,
busy, no answer, or rejection.

## 6. PHI, HIPAA mode, and the budget — BLOCKING for real callers
The intake collects medications, doses, doctors, and a pharmacy, flowing through
Vapi, Twilio, and eventually n8n. See `docs/compliance.md` for the underlying
question of whether this is PHI — that one needs a compliance contact, not a
developer.

What research settled is what it *costs* if the answer is yes. Vapi's HIPAA mode
requires a signed BAA (`security@vapi.ai`), needs Enterprise or a paid add-on at
a reported **~$2,000/month**, is org-wide with no per-assistant exception, limits
access to call logs and transcripts, restricts which LLM/TTS/STT providers can be
used, and **does not store structured outputs by default** — which is the phase 5
n8n plan.

`docs/decisions.md` sets a **~$2k/month ceiling for the entire system.** HIPAA
mode alone could consume all of it. That's a real collision and it can't be
deferred: it changes the budget, the language roadmap, and phase 5's design.

## 7. End-to-end English test
Has the full script ever been run start to finish on a real call, through the
transfer? Until that's done, everything downstream is built on an assumption.
