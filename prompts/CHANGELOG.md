# Prompt Changelog

## 2026-08-25e (SOA agent identity)
- Set the deployment value of `soa_agent_name` to `James Saint-Just`.
- Set `soa_agent_phone_spoken` to "five six one, two four seven, one four four
  three" for canonical number `+15612471443`. Both SOA identity variables now
  require a live rendering test before deployment.

## 2026-08-25d (SOA and current Vapi transfer behavior)
- Replaced the informal product-permission question with a complete recorded SOA
  script containing the date, assigned licensed-agent contact information,
  product scope, no-obligation statement, no Medicare-status impact statement,
  and no-automatic-enrollment statement.
- Added required `soa_agent_name` and `soa_agent_phone_spoken` deployment
  variables. The assistant must transfer rather than read an incomplete SOA if
  either is missing.
- Proxy callers no longer grant the beneficiary's SOA; the licensed agent must
  confirm permission and scope directly.
- Updated transfer documentation for current assistant-based warm transfer:
  operator summary plus explicit accept/cancel behavior and optional return to
  N16 after a failed transfer.

Newest first. Note *why*, not just what.

## 2026-08-25c (rewritten as a paste payload; voice-agent best practices)

**The file stopped being a document about the prompt and became the prompt.**
It had been both at once: a markdown doc with a version header, dev asides, and
repo paths, that was also pasted into Vapi verbatim. Everything below follows
from fixing that. See `prompts/README.md` for the rule going forward.

Removed from the model's context, because it was never addressed to the model:

- `## # IDENTITY` and friends — a copy artifact that was going in as literal text.
- Pointers to `docs/open-questions.md` and `config/vapi-settings.md`. The model
  cannot open them. Telling an agent about files it can't see is noise at best.
- The "Not yet verified: Haitian Creole transcription quality" note and the
  "what happens after this is out of your hands" block. Both are true, both are
  for humans, neither told the agent what to *do*.
- All markdown emphasis and blockquotes from lines meant to be spoken. The prompt
  instructs the agent never to speak formatting, then handed it
  `*Para español, dígame español.*` to read.
- `[name]` and `[ZIP]` bracket placeholders. Under load a model eventually says
  "bracket name" out loud. Substitution is described in prose now.

Added, as voice-agent practice:

- **An `ACTING VERSUS TALKING` section.** If the agent says it's doing something,
  it must call the tool in that same turn. Announcing an action and never taking
  it is the most common voice-agent failure — and it is the exact bug this
  project hit before the transfer tool existed. It deserved a named rule, not an
  assumption.
- **A turn-length cap.** One or two sentences per turn, greeting and summary
  excepted. Nothing had bounded response length; long turns hurt both latency and
  comprehension for the exact callers this serves.
- **A ban on speaking internal vocabulary.** Instructions like "record it as
  unsure" leak — the agent says "I'll record that as unsure" out loud. What is
  said and what is tracked are now separated explicitly.
- **An explicit statement of the tool inventory** (two tools, nothing else), so
  the agent can't reason its way toward a capability it doesn't have.
- Reframed voicemail handling. **These are inbound calls — there is no voicemail
  to reach.** The real case on a business line is a robocall, a dialer, or dead
  air. The old wording was an outbound-calling assumption that didn't apply.

## 2026-08-25b (branching logic formalized)

Wrote the flow down as a spec (`flows/branching-logic.md`) instead of leaving it
implicit in prose. Doing that surfaced five real gaps, all now fixed in the
prompt:

- **Added an immediate-transfer path for "let me talk to a person."** The prompt
  had no such path at all — a caller asking for a human would have gotten the
  next intake question. That single failure mode undoes the entire pitch of the
  product, so it's now a top-priority interrupt that transfers on the spot with
  a partial record.
- **Added the "neither" branch to program status.** The flow handled dual,
  Medicare-only, and Medicaid-only, but a caller enrolled in neither — someone
  turning 65 and shopping ahead, a completely ordinary caller — fell through with
  no follow-up.
- **Added escalation on repeated AI frustration.** Saying the reassurance line a
  second time to someone who already rejected it reads as stonewalling. Second
  occurrence now transfers.
- **Restored the "best time to reach you" question** (step 14). It had been
  dropped in an earlier merge. It's the field the agent works the callback from
  when a transfer fails, which is exactly the scenario the fallback plan admits
  is likely.
- **Fixed the summary correction loop.** "Correct anything they flag" was
  ambiguous enough to re-read the entire summary. Now: fix the one item, read
  back only that item.
- Also: explicit handling for a caller who declines to give a name (drop the
  name, don't substitute "sir"/"ma'am"), and unsure-at-program-status now
  explicitly asks no follow-up.

## 2026-08-25 (transfer fallback)
- Settled what happens when the transfer fires and no agent picks up: warm
  transfer, destination on a Twilio ring group rolling to voicemail, plus a
  fallbackPlan message.
- Added a note under step 14 warning against closing lines that assume the
  transfer succeeds. Vapi does not hand control back after a failed transfer,
  so "they'll be right with you" is a promise the agent can't keep.
- Corrected an earlier plan that had the AI resume and collect a callback time
  on transfer failure. Vapi doesn't support that.

## 2026-08-25 (language)
- Added trilingual opening. Spanish and Haitian Creole offers are now spoken in
  their own languages, so a caller who speaks neither English nor the other can
  still recognize their option. English-only offers were useless to the exact
  callers they were meant for.
- Expanded the LANGUAGE section into full switch-and-stay behavior.
- Flagged Haitian Creole transcription as unverified.

## 2026-08-19 (late)
- `transfer_to_licensed_agent` replaced `end_intake_call` as the normal ending.
  The handoff is now real, so the close says so.
- Added the declined-transfer path: offer a human follow-up, capture preferred
  day/time, confirm callback permission.
- Restored everything a rollback draft had dropped: name capture, the HOW YOU
  SOUND section, silence/voicemail handling, repositioned phone confirmation,
  the timing question, and dual-eligible branching.
- Split doctors / medications / pharmacy back into three separate questions.
  One combined question is too much to hold in your head at 74 while on hold.

## 2026-08-19 (earlier)
- Moved phone number capture earlier, ahead of ZIP.
- Added: employer coverage, timing (start soon vs. planning ahead), veteran
  status with a thank-you line.
- Added program-status branching: dual-eligible, Medicare-only, Medicaid-only
  each get a different follow-up.
- Consent decline no longer ends the call. Inbound beneficiary-initiated calls
  can proceed; the agent just stops naming plan types.
- Added the HOW YOU SOUND and HANDLING REAL CALLS sections after the script read
  too robotic.

## Baseline
- Five-question intake: ZIP, program status, enrolled vs. first-time,
  doctors/meds/pharmacy, cost priority. Plus AI disclosure and handoff.
