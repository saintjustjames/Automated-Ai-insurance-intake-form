# Intake Agent — Branching Logic

The authoritative map of every branch in the call. `prompts/system-prompt.md` is
what gets pasted into Vapi; **this file is why it says what it says.** If the two
disagree, this file is the spec and the prompt is the bug.

Machine-readable version: `flows/intake-flow.json`.
Fields captured: `flows/intake-schema.json`.

---

## How to read this

- **Nodes** are numbered to match the steps in the system prompt.
- **Global interrupts** can fire at *any* node and preempt whatever is in flight.
- A node is either **blocking** (the call cannot proceed past a certain answer)
  or **non-blocking** (any answer, including refusal, continues the flow).
- Only three things end the call: `transfer_to_licensed_agent`,
  `end_intake_call`, and a global interrupt that routes to one of them.

**Design rule that governs the whole flow:** almost nothing is blocking. The
caller is already on hold waiting for a human. Every refusal, "I don't know,"
and skip is recorded as-is and the flow continues. There is exactly one blocking
gate (G0, the greeting), and one gate that changes behavior without stopping
anything (N6, consent).

---

## Global interrupts

Evaluated on every caller turn, before node logic. Listed in priority order —
the first match wins.

| # | Trigger | Action | Terminal? |
|---|---|---|---|
| GI-1 | Caller describes a medical emergency (chest pain, trouble breathing, urgent symptom) | Stop intake immediately. "Please hang up and call 911 right now." No triage, no further questions. | Yes → `end_intake_call` |
| GI-2 | **Not a real caller** — automated system, recorded message, dialer, or dead air | Do not run any part of the intake. | Yes → `end_intake_call` |
| GI-3 | Wrong number / "I didn't call about insurance" | Brief apology, no intake. | Yes → `end_intake_call` |
| GI-4 | **"Just let me talk to a person"** — explicit request for a human, at any point | Do not push back, do not finish the intake. "Of course — let me get you over to a licensed agent right now." Send whatever was collected so far. | Yes → `transfer_to_licensed_agent` |
| GI-5 | Caller wants to stop / hang up | Early-exit path (N16). | Yes → `end_intake_call` |
| GI-6 | Silence | 1st (~6s): "Take your time, I'm still here." 2nd: "Are you still with me?" 3rd: close politely. | 3rd only |
| GI-7 | Asks which plan is best / what it costs / whether they qualify | "That's exactly what your licensed agent will walk you through." Return to the node that was in flight. | No |
| GI-8 | Confused about who they're speaking to | Re-disclose plainly: "I'm an AI assistant — a real licensed agent will be with you shortly." Return to node. | No |
| GI-9 | Frustrated at talking to an AI | "I completely understand — a licensed agent will be right with you. I'm just saving you from repeating everything twice." Return to node. **If they say it twice, treat as GI-4 and transfer.** | No (2nd time: yes) |
| GI-10 | Unclear audio | "Sorry, I didn't quite catch that — one more time?" Never guess. Re-ask the same node, max 2 retries, then record `unclear` and move on. | No |
| GI-11 | Rambling past the question | Let them finish — never cut off. Then: "That's really helpful. Let me get through a couple more so your agent's all set." Return to node. | No |
| GI-12 | Caller speaks or requests Spanish / Haitian Creole | Switch fully to that language for the entire remainder. Do not mix. Resume at the current node in the new language. | No |

**GI-4 is the one most easily missed.** A caller who asks for a human and gets
three more intake questions is the single worst outcome this system can produce —
it converts a "we're saving you time" pitch into "we're keeping you from a
person." Transfer immediately with a partial record.

**GI-2 is not voicemail detection.** These calls are inbound only; there is no
voicemail on the other end to reach. What actually shows up on a business line is
a robocall, an auto-dialer, or dead air. The trigger is "there is no human here,"
not "an answering machine picked up" — an outbound-calling concept that doesn't
apply to this system.

**Every terminal interrupt must call its tool in the same turn it speaks.** The
most common failure mode in a voice agent is announcing an action and never
taking it — saying "let me get you over to an agent" and then sitting there. The
prompt carries a dedicated rule for this (`ACTING VERSUS TALKING`), because this
project already hit that exact bug once, before the transfer tool existed.

---

## Node map

### G0 — Greeting consent *(BLOCKING — the only hard gate)*

Trilingual offer, then: *"Sound okay?"*

| Answer | Route |
|---|---|
| Yes / anything affirmative | → N2 |
| No | "Totally fine — let me connect you with a licensed agent now." → `transfer_to_licensed_agent` in the same turn |
| Responds in Spanish or Creole | GI-12 language switch, repeat the AI disclosure and consent question in that language, then remain at G0 until yes/no |
| Silence | GI-6 ladder |

> Blocking on purpose. Running an intake on someone who declined it is the one
> consent failure with actual teeth.

---

### N2 — Name *(non-blocking)*

*"Who do I have the pleasure of speaking with?"*

- Given → store `caller_name`, use first name every 3rd–4th exchange from here.
- Declined → store `null`, drop all `[name]` interpolation for the rest of the
  call. Do **not** substitute "sir," "ma'am," or a guessed honorific.

---

### N3 — Who it's for *(routing flag, non-blocking)*

*"Is this coverage for you, or are you helping someone else out?"*

| Answer | Effect |
|---|---|
| Self | `is_proxy = false`. Default path. |
| Helping someone else | `is_proxy = true`. Capture `beneficiary_name`, `proxy_relationship`, and `beneficiary_present`, one question at a time. **Every subsequent clinical/coverage answer describes the beneficiary, not the caller.** Phone and callback consent still describe the *caller*. |
| Unclear | Ask once more, then default to self and flag `unclear` for the agent. |

> The proxy split is a data-labeling branch, not a wording branch. The questions
> stay the same; what changes is who the recorded answers are *about*. Getting
> this wrong hands the agent a medication list attached to the wrong person.

---

### N4 — Phone *(non-blocking, 3 sub-questions)*

1. *"What's the best number to reach you at?"* → read back digit by digit.
   - Confirmed → store. Corrected → re-read back, loop max 2 times.
   - Declined → store `null`, **skip 4.2 and 4.3 entirely** (no number to consent
     to a callback on).
2. *"If we get disconnected, is it alright if someone calls you back at that
   number?"* → `callback_consent` yes/no. **Never blocks anything.**
3. *"Any other number you'd want your agent to have?"* → optional
   `alt_phone`, read back if given.

---

### N5 — ZIP or county *(non-blocking)*

- Valid 5-digit ZIP → read back digit by digit, confirm, store.
- County name instead → accept it, store as `county`, do not press for a ZIP.
- Unclear → GI-10 retry ladder (max 2), then `unclear`.
- Declined → store `null`, and in N6 **drop the "since you're in [ZIP]" clause**
  rather than saying it with a hole in it.

---

### N6 — Recorded Scope of Appointment *(BEHAVIOR GATE — does not block intake)*

The SOA must be read verbatim and must identify the licensed agent assigned to
the call. `soa_agent_name` and `soa_agent_phone_spoken` are required Vapi
variables; **do not deploy this node with either value missing**.

For a caller seeking coverage for themself, read:

*"Before any plan options are discussed, I need to document the scope for
today's call. Today is [the current date]. You're agreeing that
[soa_agent_name], a licensed agent with Prep and Seal Insurance who can be
reached at [soa_agent_phone_spoken], may discuss Medicare Advantage,
prescription drug plans, Medicare Supplement, and hospital indemnity or other
supplemental plans with you. There's no obligation to enroll, your current or
future Medicare enrollment status won't be affected, and you won't be enrolled
automatically. Do you agree to this scope?"*

The bracketed values above describe runtime substitutions; they must never be
spoken literally. The live prompt uses Vapi Liquid variables instead.

If `is_proxy = true`, do not ask the proxy to grant the beneficiary's SOA. Say
the licensed agent will confirm the required permission and scope, then transfer
with the partial intake. Do not continue into plan-related questions.

| Answer | Effect |
|---|---|
| Yes | `soa_consent = granted`. Store the audit fields below. Normal path. |
| No **or** unsure | `soa_consent = declined` / `unsure`. "No problem at all — I won't discuss plan options. I can still connect you with a licensed agent." **Set `plan_naming_suppressed = true` and route to N14.** Skip N7–N13; the licensed agent must obtain a valid SOA before any scoped product discussion. |

For every attempt, store `soa_script_version`, `soa_timestamp`,
`soa_product_scope`, `soa_agent_name`, `soa_agent_phone`, `soa_response_verbatim`,
and whether the caller was the beneficiary. The call recording remains the
primary evidence.

**What suppression actually means:** from this point the agent must not name a
plan type, Medicare/Medicaid program, Medicare Part, product line, benefit, or
coverage qualifier. N7–N13 are skipped. Only neutral callback timing, a short
summary of already collected contact facts, and transfer remain.

> Declining consent does not end the call. These are inbound,
> beneficiary-initiated callers already waiting for an agent. See
> `docs/compliance.md`.

---

### N7 — Program status *(THE MAIN BRANCH)*

*"Are you enrolled in Medicare, Medicaid, or both?"*

| Answer | `program_status` | Follow-up |
|---|---|---|
| Both | `dual` | "Got it — that's good to know, it opens up options your agent will want to go over." Then: *"Do you know roughly when your Medicaid was last renewed?"* → `medicaid_renewal_timing`, accept "not sure." |
| Medicare only | `medicare_only` | *"Do you have Part A and Part B, or just one?"* → `medicare_parts` in {A, B, both, unsure}. |
| Medicaid only | `medicaid_only` | *"Are you coming up on Medicare eligibility soon, or not yet?"* → `medicare_eligibility_timing`. |
| **Neither** | `neither` | *"Got it — are you coming up on Medicare eligibility soon?"* → `medicare_eligibility_timing`. **(Turning-65 shopper. Previously unhandled — the flow had no path for a caller in neither program.)** |
| Unsure | `unsure` | **Ask no follow-up.** Record unsure and move to N8. |

**Hard rules on this node:**
- Never guess which program someone is in.
- Never explain the difference between Medicare and Medicaid. That is plan
  education, and it is the licensed agent's job.
- "Not sure" is a valid, complete, recordable answer at every level here.

> This branch is the whole reason the script isn't flat. A dual-eligible caller
> and a turning-65 caller need visibly different conversations, and the agent
> picking up the transfer needs to know which one is on the line before they say
> hello.

---

### N8–N11 — Straight-line qualifiers *(all non-blocking, any answer continues)*

| Node | Question | Field |
|---|---|---|
| N8 | "Coverage through an employer right now — yours, a former employer, or a spouse's?" | `employer_coverage` |
| N9 | "Currently in a plan, or first time shopping?" | `shopping_status` |
| N10 | "Looking for something to start soon, or planning ahead for next year?" | `timing` |
| N11 | "Are you, or your spouse, a veteran?" | `veteran_status` |

**N11 branch:** if yes → *"Thank you for your service — and to your spouse as
well, if that applies."* Then continue either way. This is a courtesy line and an
eligibility signal (VA benefits coordination) — it is **not** a qualifying gate,
and nothing downstream changes based on it.

---

### N12 — Doctors, medications, pharmacy *(three separate questions)*

Preamble: *"No wrong way to answer — whatever you remember is fine. If a
medication's hard to pronounce, just spell it or tell me what it's for."*

Then, one at a time, waiting for a complete answer between each:

1. *"Who are the doctors you see regularly?"* → `doctors[]`
2. *"What medications are you taking these days? If you know the dose or
   directions, include them — no worries if not."* → `medications[]`
   (`{name, dose_verbatim?, strength_value?, strength_unit?, frequency?, verbatim}`)
3. *"Which pharmacy do you use?"* → `pharmacy`

**Per-medication spell-back loop:** if a name isn't clearly heard, spell it back
and confirm. Max 2 attempts, then record the caller's verbatim transcription with
`confidence: low` and move on. **Never** synthesize a plausible drug name from a
partial — a wrong medication in the record is worse than a blank.

Never comment on what a medication treats or whether it would be covered.

Each of the three is independently skippable.

> Split into three because one combined question is too much to hold in your head
> at 74 while on hold. See `prompts/CHANGELOG.md`, 2026-08-19.

---

### N13 — Cost priority *(non-blocking)*

*"Is it more important to keep your monthly premium low, or keep your costs down
when you actually go to the doctor?"*

- Premium / out-of-pocket / both / unsure → `cost_priority`
- Volunteered extras (dental, vision, travel, a specific drug) → `other_concerns[]`

---

### N14 — Best time to reach *(non-blocking)*

*"What's generally the best time of day to reach you?"* → `best_time_to_reach`

Matters more than it looks: it is what the agent uses if the transfer at N15
fails and the voicemail queue has to be worked.

---

### N15 — Summary and transfer

Read back **essentials only** — caller/beneficiary identity, ZIP or county,
program status, medication count, any low-confidence medication requiring human
confirmation, and cost priority. Do not read the full medication list unless the
caller asks. Under ~15 seconds. Then *"Does that all sound right?"*

| Answer | Route |
|---|---|
| Confirmed | → transfer |
| Correction | Update the named field only. **Re-read just the corrected item, not the whole summary.** Loop max 2 times, then accept and move on. |

Then: *"Perfect, [name] — you did great. Let me get you over to a licensed agent
now. They'll have everything we just went through."* → `transfer_to_licensed_agent`

Assistant-based warm transfer holds the caller while the transfer assistant
speaks to the destination. A human acceptance calls `transferSuccessful`.
Voicemail, busy, no answer, or rejection calls `transferCancel`. With
`fallbackPlan.endCallEnabled: false`, control returns to this assistant and it
runs N16. An initiated transfer is not counted as successful until a human
accepts.

---

### N16 — Early exit / declined transfer

Reached from GI-5, or from a caller who declines the transfer at N15.

*"Absolutely, no problem. Would you like a licensed agent to follow up with you
another time?"*

| Answer | Route |
|---|---|
| Yes, phone exists | Ask preferred day/time → `callback_preference`. If consent was not granted for that exact number, ask now. Then end without claiming the callback is already scheduled. |
| Yes, phone is null | Ask for a callback number, read it back and confirm, then ask preferred day/time and callback consent for that exact number. If no number is provided, explain that callback cannot be arranged and end. |
| No | "Understood. Thanks for calling, [name]." → `end_intake_call` |

Never push back. One offer, then respect the answer.

---

## Known gaps

1. **Post-transfer failure now depends on correct transfer-assistant actions.**
   `transferCancel` plus `fallbackPlan.endCallEnabled: false` returns the caller
   to N16. Missing or incorrect accept/cancel rules can still connect voicemail
   or strand the caller. Test the destination leg, not only the initial transfer
   event. *(`docs/open-questions.md` #1)*
2. **GI-12 language switch is unproven for Haitian Creole.** If transcription
   can't hold Creole, GI-12 should route Creole callers straight to GI-4
   (immediate human transfer) rather than attempt an intake in a language the
   system can't reliably hear. *(`docs/open-questions.md` #2)*
3. **No caller-identity verification.** The flow trusts self-reported identity
   entirely. Fine for intake; would need revisiting before any record-lookup
   phase (roadmap phase 7).
