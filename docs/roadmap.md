# Roadmap

In order. Don't skip ahead.

## Phase 1 — Prove English (current)

**Do the first item first.** Warm transfer is documented as Twilio-only and the
assistant is currently on a Vapi-native number. Testing transfer behavior before
fixing that means debugging a limitation instead of removing it.

- [ ] **Import the Twilio number into Vapi and assign this assistant to it.**
      Blocks every transfer test below.
- [ ] Set `voicemailDetectionType` on the transfer plan, and confirm the
      voicemail-detection provider is one Vapi supports for this mode.
- [ ] Configure assistant-based `warm-transfer-experimental` with a concise
      operator summary, explicit `transferSuccessful` / `transferCancel` rules,
      and `fallbackPlan.endCallEnabled: false` so N16 can run after failure.
- [ ] Run a full live test call, greeting through transfer.
- [ ] Confirm `transfer_to_licensed_agent` actually reaches a human.
- [x] Decide the no-agent-available fallback. Warm transfer + voicemail.
- [ ] Verify warm transfer **actually holds** before connecting, on the Twilio
      number. Known to silently behave as a blind transfer.
- [ ] Set up the Twilio ring group and voicemail box the transfer lands on.
- [ ] Assign someone to actually work that voicemail queue.
- [ ] Tune voice preset and speed; confirm barge-in is on.
- [ ] Apply `config/vapi-noise-controls.json`: Smart Denoising on, Fourier off,
      `numWords: 2`, critical one-word `interruptionPhrases`, 0.8-second response
      wait, and background ambience off. Verify with scenarios 22a–22d.
- [ ] Re-paste the rewritten prompt into Vapi (it changed materially — see
      `prompts/CHANGELOG.md` 2026-08-25c).
- [ ] Run the **P0 block** in `tests/call-scenarios.md`. Blocks go-live.
- [ ] Resolve PHI / BAA coverage with a compliance contact. Blocks real callers.
- [ ] Configure and counsel-approve `recordingConsentPlan`; document the
      recording start boundary, refusal path, retrieval, and retention proof.
- [ ] Obtain carrier approval for the exact N6 SOA script and determine whether
      a TPMO verbal disclaimer is required.
- [ ] Set silence timeout and max call duration explicitly.
- [ ] Move the greeting to `firstMessage` and delete step 1's spoken line from
      the prompt in the same change.
- [ ] Wire `flows/intake-schema.json` up as the structured-output schema — phase
      5 depends on it and it's cheaper to do now than to retrofit.
- [ ] Evaluate prompt caching using actual cache-write/read pricing and observed
      hit rate. Enable it only when logs show a net saving.
- [ ] Try a cheaper model, then re-run P0 plus scenarios 3, 4, 19, 27. Keep it
      only if it passes clean. See `docs/cost-model.md`.

## Phase 2 — Spanish
- Duplicate the proven English assistant.
- Translate the intake natively — not machine-translated, written to sound right.
- Swap in a Spanish voice.

## Phase 3 — Haitian Creole
- Only after transcription is verified. See open-questions #2.
- Same duplicate-and-translate pattern.

## Phase 4 — Squad
Analysis and risks: `docs/multi-assistant.md`. Squads, **not** Workflows — Vapi
no longer recommends Workflows for new builds.

- Lightweight greeter assistant handles the language offer and routes.
- Hands off to the dedicated EN / ES / HT assistants from phases 1–3.
- **Replicate the global interrupts in every assistant** — emergency stop,
  immediate transfer on a request for a human, frustration escalation. An
  assistant missing them will ask about a pharmacy while someone describes chest
  pain.
- **Attach the transfer tool to every assistant**, not just the closer.
- **Verify consent and proxy flags survive each handoff.** `plan_naming_suppressed`
  crossing a boundary incorrectly is a compliance failure; `is_proxy` crossing
  incorrectly attributes a medication list to the wrong person.
- Add a handoff-boundary test per assistant: one emergency and one "let me talk
  to a person" fired from *each* one, plus a state-carryover check.
- Optional once stable: per-assistant model tiers — cheap for mechanical capture,
  capable for branching, clinical, and close.

## Phase 5 — Post-call automation
- **Wire up structured outputs first.** Both designs below depend on it.
- Decide: webhook receiver (n8n) vs. scheduled agent polling the Vapi API.
  See `automation/README.md`. Note that HIPAA mode, if enabled, doesn't store
  structured outputs by default — which breaks the polling design.
- Log the call summary and structured intake data.
- Push to a CRM or calendar so agents can track and follow up.
- **Create the follow-up task for failed transfers.** This is the other half of
  the promise the fallback message makes to the caller.

## Phase 6 — Plan availability lookup
- Tie the caller's ZIP to which plans are actually available there, and let the
  agent reference it live.
- Explicitly deferred as its own build phase. The greeting line about "plans
  available in your area" stays out until this exists.

## Phase 7 — IntegrityCONNECT integration
- Pull member/plan records from Ritter/Integrity ahead of the agent call.
- The only piece that needs real developer work.
