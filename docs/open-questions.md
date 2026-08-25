# Open Questions

Answer these before building past them.

## 1. Transfer with no agent available — DECIDED, needs testing
**Resolved 2026-08-25.** Warm transfer with a fallbackPlan message, destination
pointed at a Twilio ring group that rolls to voicemail. See `docs/decisions.md`.

Still open, and it's a real risk:
- Multiple users have reported warm transfer connecting immediately instead of
  honoring the configured mode. **Test it. Don't trust the setting.**
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

## 5. PHI and vendor BAAs — BLOCKING for real callers
The intake collects medications, doses, doctors, and a pharmacy. That data flows
through Vapi, Twilio, and eventually n8n. No BAA with any of them is mentioned
anywhere in this project. See `docs/compliance.md` — needs a compliance contact,
not a developer, and needs answering before the number takes live callers.

## 6. End-to-end English test
Has the full script ever been run start to finish on a real call, through the
transfer? Until that's done, everything downstream is built on an assumption.
