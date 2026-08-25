# Compliance Notes

Not legal advice. These are the constraints the script was built around.

## Scope of Appointment
- The 48-hour SOA waiting period **does not apply** to beneficiary-initiated
  inbound calls. These callers are inbound and on hold, so it doesn't bind here.
- CMS is removing the 48-hour rule entirely as of **October 1, 2026** — verified
  against the CY2027 final rule (published April 2026), which eliminates the
  waiting period at 42 CFR 422.2264(c)(3) and 423.2264(c)(3).
- **It is still in effect right now.** Until Oct 1, 2026 the 48-hour rule still
  binds *scheduled* personal marketing appointments. This intake flow is
  unaffected either way, because it runs on inbound beneficiary-initiated calls
  — but don't let the Oct 1 date get read as "the rule is already gone" for the
  rest of the agency's business.
- After Oct 1, a valid SOA is **still required** before any plan-specific
  discussion. What changes is only the waiting period: SOA and appointment may
  happen in the same call. That is exactly what step 6 of the script does.

Sources: [Hall Render](https://hallrender.com/2026/06/01/cms-revises-medicare-advantage-marketing-guidance-for-scope-of-appointment-forms/),
[PSM Brokerage](https://www.psmbrokerage.com/blog/the-48-hour-scope-of-appointment-rule-is-gone-what-medicare-agents-need-to-know),
[Lourie Agents](https://www.lourieagents.com/cms-removes-the-48-hour-soa-waiting-period-what-agents-need-to-know-for-cy-2027/).
Have your compliance contact confirm before relying on any of it.
- Consent is still collected verbally on the call, covering the full product
  line: Medicare Advantage, prescription drug plans, Medicare Supplement, and
  hospital indemnity / other supplemental.
- Declining consent does not end the call. The intake continues; the agent stops
  naming plan types.
- Callback consent is captured separately and never blocks anything.

## AI disclosure
Disclosed in the opening line, every call. Re-disclosed any time the caller
seems unsure who they're talking to.

## Recordings
CMS-required call recordings and retention are handled at **Ritter**. Nothing is
stored in house.

## Protected health information — UNRESOLVED

Step 12 collects a medication list, doses, treating physicians, and a pharmacy.
That is health information tied to an identified individual, gathered by an
entity arranging health coverage. Treat it as PHI until someone qualified tells
you otherwise.

Which means it is not only Ritter that holds sensitive data. The call audio, the
live transcript, and the extracted intake record pass through **Vapi**, the
telephony leg through **Twilio**, and — once phase 5 exists — the structured
record through **n8n** and whatever it writes into.

Before this number takes real callers, confirm with a compliance contact:

- Whether the agency is a covered entity or business associate here.
- Whether a **BAA** is needed with Vapi, with Twilio, and with the n8n host, and
  whether each will sign one on the current plan. Not every vendor tier offers
  one.
- How long transcripts and recordings persist at each vendor, and whether that
  retention is compatible with the CMS retention already handled at Ritter.
- Where the phase 5 record lands, and whether that destination is in scope.

This is flagged, not resolved. Nobody in this repo is qualified to answer it, and
it is the kind of gap that is far cheaper to close before launch than after.

## Hard boundaries in the prompt
- No plan or carrier names.
- No coverage or eligibility determinations.
- No pricing.
- No medical advice. Emergencies → hang up and call 911, no triage.
- No confirming a medication name or dose the caller didn't clearly say.
