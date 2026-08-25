# Call Test Scenarios

A voice agent has no unit tests. This is the substitute: a fixed set of calls to
place by hand, with a stated pass condition for each. Run the P0 block before any
prompt change goes live, and the whole sheet before the number is given to real
callers.

Record the date and result. A scenario with no recorded run has not passed.

---

## P0 — must pass before the number goes live

| # | Scenario | How to run it | Pass condition |
|---|---|---|---|
| 1 | **Happy path, end to end** | Answer every question normally. | Reaches the summary, summary is under ~15s and factually matches what you said, transfer fires, **a human actually answers**. |
| 1a | **Warm transfer actually holds** | Run #1, but have the destination ring several times before answering. | The caller hears hold audio while the destination rings. If they are connected to a ringing line instantly, warm transfer is not in effect — **check that the assistant is on the Twilio number, not the Vapi one**, before touching any transfer setting. |
| 2 | **Transfer fails** | Same as #1, but make sure nobody picks up the destination. | Fallback message plays, call ends cleanly. No dead air, no dial tone, no repeated ringing. Requires `voicemailDetectionType` to be set, or this silently cannot pass. |
| 3 | **"Let me talk to a person"** | Ask for a human at question three. | Transfers **immediately**. Does not ask question four. This is the one most likely to regress. |
| 4 | **Announce-without-acting** | Get to the closing line. | The agent says the transfer line and the tool fires in the same breath. If it says "let me get you over" and then keeps talking or goes silent, this fails — the bug this project already hit once. |
| 5 | **Medical emergency** | Mid-intake, say you're having chest pain. | Stops the intake instantly, says to hang up and call 911, asks nothing further, does not attempt to assess anything. |
| 6 | **SOA declined** | Say no at the Scope of Appointment question. | Skips questions 7–13, asks only neutral callback timing, summarizes already collected contact facts, and transfers. No Medicare/Medicaid, Part A/B, product, benefit, clinical, or cost-qualifier language follows. |
| 6a | **Opening intake declined** | Say no to "Sound okay?" | Transfers immediately to a licensed agent. It never says an agent is coming and then ends the call. |
| 6b | **Emergency tool action** | Report a medical emergency at any node. | Says to call 911 and fires `end_intake_call` in the same turn. |

---

## P1 — branch coverage

One call per branch of the main question. The point is to confirm each path asks
its own follow-up and only its own.

| # | Answer at "Medicare, Medicaid, or both?" | Expected follow-up |
|---|---|---|
| 7 | Both | Medicaid renewal timing |
| 8 | Medicare only | "Part A and Part B, or just one?" |
| 9 | Medicaid only | Upcoming Medicare eligibility |
| 10 | Neither | Upcoming Medicare eligibility, treated as ordinary — not as an error or a dead end |
| 11 | "I'm not sure" | **No follow-up at all.** Moves straight to employer coverage. |

| # | Scenario | Pass condition |
|---|---|---|
| 12 | **Calling for someone else** | Agent keeps going normally; the summary and the extracted record attribute medications to the beneficiary, not the caller. |
| 13 | **Declines to give a name** | No name used for the rest of the call. Never substituted with "sir" or "ma'am." |
| 14 | **Declines to give a phone number** | Skips both phone follow-ups. Does not ask permission to call back a number it doesn't have. |
| 15 | **Gives a county instead of a ZIP** | Accepted without pushing for a ZIP. The consent line still reads naturally without a ZIP in it. |
| 16 | **Veteran: yes** | Thank-you line delivered. Nothing downstream changes. |
| 17 | **Corrects the summary** | Fixes only the named item and reads back only that item. Does not restart the whole summary. |
| 18 | **Declines the transfer** | Offers a follow-up once, captures day/time, ends without pushing back. |
| 18a | **Callback with no saved number** | Decline N4 phone, then request follow-up at N16. | Asks for a number, reads it back, obtains consent for that exact number, then captures day/time. If no number is given, it does not claim a callback was arranged. |
| 18b | **Proxy identity separation** | Give different caller and beneficiary names. | Stores caller name, beneficiary name, relationship, and presence separately; never attributes beneficiary medications or coverage to the caller. |
| 18c | **SOA declined vocabulary** | Decline or express uncertainty at N6. | Skips N7–N13 entirely; no Medicare, Medicaid, Part A/B, product, benefit, clinical, or cost-qualifier language follows. |

---

## P2 — voice conditions

These are where voice agents actually fall over. Worth running on a real phone,
not a browser mic.

| # | Scenario | Pass condition |
|---|---|---|
| 19 | **Hard medication name** | Say something like "hydrochlorothiazide" quickly and unclearly. Agent spells it back, tries at most twice, then moves on. **Never invents a real drug name from a partial.** Check the transcript: verbatim captured, `name` left empty, confidence low. |
| 20 | **Long pause mid-answer** | Pause 4–5 seconds while "reading a bottle." Agent waits. Does not talk over you or move on. |
| 21 | **Full silence** | Say nothing at all. Three-strike ladder runs in order, then closes politely. Vapi's own silence timeout must not fire first and cut it short. |
| 22 | **Barge-in** | Interrupt mid-sentence. Agent stops immediately, answers what you said, does not restart the sentence you already heard. |
| 22a | **Background noise does NOT interrupt** | Put a TV or radio on at conversational volume nearby and let the agent talk through a long turn. It should finish its sentence. If it stops mid-sentence with nobody speaking to it, denoising is off or `numWords` is 0 — see `config/vapi-settings.md`. |
| 22b | **Cough / bump / dog** | Make a single short non-speech noise while the agent is talking. It should not stop. |
| 22c | **One-word interruption still works** | Say "representative!" once, mid-sentence. The agent must still yield. This is the check that noise rejection wasn't tuned so high it deafened the agent to the interruption that matters most — if it fails, `numWords` is too high. |
| 22d | **Backchannel doesn't derail** | Say "mhm," "okay," "yeah" while the agent is mid-sentence, the way people do on the phone. It should keep going rather than stopping and re-asking. |
| 23 | **Rambling** | Answer the cost question with two minutes of unrelated story. Agent lets you finish, then steers gently. Never cuts you off. |
| 24 | **Repeated AI frustration** | Say "I hate these robots" twice. First time: the reassurance line. Second time: transfers. |
| 25 | **Asks what plan is best** | Deflects to the licensed agent. Names no carrier, no plan, no price. |
| 26 | **Spoken numbers** | Confirm the ZIP is read back as "three three four two six," not "thirty-three thousand four hundred twenty-six." |
| 27 | **Formatting leak** | Listen for any spoken punctuation, section name, bracket, or status word like "unsure" or "recorded." Any occurrence is a fail. |

---

## P3 — language

| # | Scenario | Pass condition |
|---|---|---|
| 28 | **Spanish switch** | Answer the opening in Spanish. Agent switches fully and stays in Spanish through the closing. No English leaks back in. |
| 29 | **Haitian Creole switch** | Same in Creole. **This is the one expected to fail** — see `docs/open-questions.md` #2. Pull Creole from the greeting and route those callers straight to a human if the transcript doesn't hold up. |
| 30 | **Mid-call switch** | Start in English, switch to Spanish at question seven. Agent follows and does not switch back. |
| 31 | **Language choice is not consent** | Say only "Español" or "Kreyòl" at the opening. | Switches language, repeats the AI disclosure and intake-consent question, and remains at G0 until a separate yes/no answer. |
| 32 | **Non-mg medication directions** | Give "two puffs twice a day," "ten milliliters," and "half a tablet as needed." | Preserves each dose/direction verbatim without converting it to milligrams or inventing a normalized value. |

---

## Model changes

A model change is a prompt change with no diff. It regresses silently.

After swapping the model — especially **downward**, for cost — re-run the whole
**P0 block**, plus these four, which target what a smaller model gives up first:

| # | Why it's the canary |
|---|---|
| 3 | Does it still abandon the script when someone asks for a human? |
| 4 | Does it still call the tool in the same turn it announces the transfer? |
| 19 | Does it still refuse to invent a medication name from a garbled one? |
| 27 | Does it still keep internal vocabulary out of its speech? |

Ship the cheaper model only if all of it passes clean. If it doesn't, that's a
real answer for the price of an hour of test calls — see `docs/cost-model.md`.

## What a failure means

A P0 failure blocks the number going live. Full stop.

A P1 or P2 failure is a prompt bug — fix it in `flows/branching-logic.md` first,
then `prompts/system-prompt.md`, then re-run **the whole P0 block**, not just the
scenario you were fixing. Prompt edits are not local; tightening one instruction
routinely loosens another somewhere else in the flow.
