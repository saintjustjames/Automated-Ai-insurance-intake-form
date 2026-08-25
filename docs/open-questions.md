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
- Vapi cannot resume the assistant conversation after a failed transfer. The
  fallback message gets one sentence, then the call ends. There is no path back
  into the script — confirm the message alone is enough.
- The fallback message promises a human will follow up. Someone has to actually
  work that voicemail queue, or the promise is a lie.

## 2. Haitian Creole transcription quality — BLOCKING for Creole
Spanish is safe. Creole is not verified. Run a live test call in Creole and see
whether the transcript holds up. If it doesn't, pull Creole from the greeting and
route those callers straight to a human instead.

Concretely, if it fails: interrupt GI-12 in `flows/branching-logic.md` should
route Creole callers to GI-4 (immediate transfer) rather than attempt an intake
in a language the transcriber can't reliably hear. A garbled Creole medication
list is worse than no medication list.

## 3. Squad vs. single assistant
Researched, not decided. Squad = a greeter assistant routes to dedicated
per-language assistants with their own voices and natively-written scripts.
Better quality, more moving parts. Deferred until English is proven.

## 4. Who works the voicemail queue?
Not a technical question, and it blocks Phase 1 anyway. The transfer fallback
message tells the caller "someone will reach out to you shortly." If nobody is
assigned to that queue, the system's failure path is a recorded lie to a senior
who was promised a callback. Name a person before the number goes live.

## 5. Transfer mode: summary to the agent, or voicemail fallback?
Vapi does not appear to offer both in one mode.

- `warm-transfer-experimental` (current): voicemail detection and a fallback
  message. The receiving agent gets **no spoken summary**.
- `warm-transfer-with-summary` / `...-wait-for-operator-to-speak-first-and-then-
  say-summary`: an AI summary read to the agent before connecting. **No voicemail
  fallback.**

The summary modes deliver the actual product promise — the agent picks up already
knowing who's on the line. The current mode protects the failure path instead.
Both matter. Someone has to choose, or find a way to stack them.

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
