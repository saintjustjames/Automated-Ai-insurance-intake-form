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
- [ ] Decide transfer mode: voicemail fallback vs. spoken summary to the agent.
      See `docs/open-questions.md` #5.
- [ ] Run a full live test call, greeting through transfer.
- [ ] Confirm `transfer_to_licensed_agent` actually reaches a human.
- [x] Decide the no-agent-available fallback. Warm transfer + voicemail.
- [ ] Verify warm transfer **actually holds** before connecting, on the Twilio
      number. Known to silently behave as a blind transfer.
- [ ] Set up the Twilio ring group and voicemail box the transfer lands on.
- [ ] Assign someone to actually work that voicemail queue.
- [ ] Tune voice preset and speed; confirm barge-in is on.
- [ ] Re-paste the rewritten prompt into Vapi (it changed materially — see
      `prompts/CHANGELOG.md` 2026-08-25c).
- [ ] Run the **P0 block** in `tests/call-scenarios.md`. Blocks go-live.
- [ ] Resolve PHI / BAA coverage with a compliance contact. Blocks real callers.
- [ ] Set silence timeout and max call duration explicitly.
- [ ] Move the greeting to `firstMessage` and delete step 1's spoken line from
      the prompt in the same change.
- [ ] Wire `flows/intake-schema.json` up as the structured-output schema — phase
      5 depends on it and it's cheaper to do now than to retrofit.

## Phase 2 — Spanish
- Duplicate the proven English assistant.
- Translate the intake natively — not machine-translated, written to sound right.
- Swap in a Spanish voice.

## Phase 3 — Haitian Creole
- Only after transcription is verified. See open-questions #2.
- Same duplicate-and-translate pattern.

## Phase 4 — Squad
- Lightweight greeter assistant handles the language offer and routes.
- Hands off to the dedicated EN / ES / HT assistants from phases 1–3.

## Phase 5 — Post-call automation (n8n)
- Log the call summary and structured intake data.
- Push to a CRM or calendar so agents can track and follow up.
- Hangs off the summary step in the flow.

## Phase 6 — Plan availability lookup
- Tie the caller's ZIP to which plans are actually available there, and let the
  agent reference it live.
- Explicitly deferred as its own build phase. The greeting line about "plans
  available in your area" stays out until this exists.

## Phase 7 — IntegrityCONNECT integration
- Pull member/plan records from Ritter/Integrity ahead of the agent call.
- The only piece that needs real developer work.
