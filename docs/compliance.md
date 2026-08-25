# Compliance Notes

Not legal advice. Carrier compliance or qualified counsel must approve the live
script, recording design, and data flow before real callers use it.

## Scope of Appointment

- CMS's 48-hour SOA waiting rule applies to scheduled personal marketing
  appointments and excludes inbound calls. That does not remove the obligation
  to agree upon and document scope before plan-product marketing.
- CMS removes the waiting period effective October 1, 2026, but retains advance
  SOA. Do not read that effective date as eliminating the SOA itself.
- N6 records the date, assigned licensed-agent contact, product categories,
  no-obligation statement, no Medicare-enrollment-impact statement,
  no-automatic-enrollment statement, and the caller's verbatim response.
- `soa_agent_name` and `soa_agent_phone_spoken` are required deployment
  variables. If either is missing, the assistant transfers rather than reading
  an incomplete scope.
- A proxy does not grant the beneficiary's SOA in this flow. The licensed agent
  confirms authority, permission, and scope directly.
- Declining or being unsure does not end the call, but N7–N13 are skipped. The
  licensed agent must obtain valid scope before product discussion.

Primary sources: [CMS CY2025 Agent/Broker Training, Q22](https://www.cms.gov/files/document/cy2025-agent-broker-training-testing-guidelines.pdf),
[CMS Medicare Communications and Marketing Guidelines, SOA elements](https://www.cms.gov/files/document/medicare-communications-marketing-guidelines-2-9-2022.pdf),
and the [CY2027 final rule](https://www.govinfo.gov/content/pkg/FR-2026-04-06/pdf/2026-06600.pdf).

## Recording notice and consent — BLOCKING

Configure Vapi's `recordingConsentPlan` so notice and affirmative consent occur
before substantive intake. Florida requires prior consent of all parties for
interception under Fla. Stat. §934.03(2)(d). Carrier compliance or counsel must
approve the exact notice, when recording begins, the refusal path, and the
multistate policy.

CMS recording and retention duties may cover the full interaction if this is a
TPMO marketing, sales, or enrollment call. Saying Ritter handles retention is
not evidence of compliance. Before launch, document the contract, full-call
capture boundary, retrieval procedure, access controls, and ten-year retention
proof. Keep required recording retention separate from shorter operational
retention for duplicate transcripts, model logs, and extracted records.

Sources: [Florida Statute §934.03](https://www.leg.state.fl.us/statutes/index.cfm?App_mode=Display_Statute&URL=0900-0999/0934/Sections/0934.03.html)
and [CMS Agent/Broker Marketing FAQs](https://www.cms.gov/files/document/agent-broker-marketing-faqs-10-19-2022.pdf).

## AI disclosure

The opening discloses that the caller is speaking to an AI assistant, and GI-8
repeats it when needed. Preserve this as a transparency practice. Do not describe
it as a complete nationwide legal conclusion; state AI rules continue to evolve.

## Sensitive health information and HIPAA status — UNRESOLVED

Step 12 collects medications, directions, treating physicians, and a pharmacy.
That is sensitive identifiable health information. It is PHI under HIPAA only
when held or transmitted by a covered entity or business associate in this
workflow; an insurance agency is not automatically either merely because it
arranges coverage. Treat the data as highly sensitive regardless.

Before launch, counsel must map the role of the agency, Ritter, each plan, Vapi,
Twilio, the model/STT/TTS providers, n8n, and the final storage system. If a
business-associate relationship exists, execute the appropriate BAAs and verify
safeguards for every PHI-handling vendor or subcontractor.

Even if HIPAA does not apply, the FTC Act and state security/breach laws still
apply to representations and safeguards around consumer health data. Define a
field-by-field purpose, permitted recipients, access roles, encryption,
retention/deletion period, and incident procedure. See [HHS covered entities](https://www.hhs.gov/hipaa/for-professionals/covered-entities/index.html),
[HHS business associates](https://www.hhs.gov/hipaa/for-professionals/privacy/guidance/business-associates/index.html),
and [FTC/HHS health-data guidance](https://www.ftc.gov/business-guidance/resources/collecting-using-or-sharing-consumer-health-information-look-hipaa-ftc-act-health-breach).

## Callback consent

Callback consent authorizes a human licensed-agent callback to the exact number
confirmed on the call for this insurance request. Store the number, purpose,
channel, timestamp, response verbatim, script version, and any revocation. It
does not authorize an automated or AI-voice callback. If automated callbacks are
added later, obtain counsel-approved TCPA language first. See [FCC 24-17](https://docs.fcc.gov/public/attachments/FCC-24-17A1.pdf).

## TPMO classification — BLOCKING

Carrier compliance must determine whether this intake is itself a TPMO
communication, marketing, or sales call and whether a verbal TPMO disclaimer is
required. That classification controls recording, retention, and disclaimer
requirements. Avoidance of carrier and specific plan names does not settle it.

## Hard prompt boundaries

- No carrier or specific plan names outside the approved SOA categories.
- No coverage or eligibility determinations.
- No pricing or medical advice.
- Emergencies instruct the caller to call 911 and fire `end_intake_call` in the
  same turn; no triage.
- No confirming a medication name, strength, unit, or direction the caller did
  not clearly state.
