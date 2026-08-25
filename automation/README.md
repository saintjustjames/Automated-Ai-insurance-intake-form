# Post-call automation

What happens to the intake record after the call ends: logging it, getting it in
front of the right agent, and creating the follow-up.

## The constraint that decides the design

**Something has to be running when a call ends.** Calls end at 7pm on a Saturday.
Whatever handles the record has to be awake then, with nobody watching.

That rules out one tempting option: **Claude Code in a terminal is not a
service.** It runs when someone opens it and types. It cannot catch a webhook
from a call that just ended, and building the plan around "Claude will handle it"
means, in practice, that nothing handles it.

That does **not** mean n8n is the only answer. It means the answer has to be
something that runs on its own.

## Two real options

### A. Webhook receiver (n8n, or any always-on endpoint)

Vapi posts the end-of-call payload to a server URL the moment the call ends. The
receiver parses it and writes it wherever it goes.

- **Latency:** immediate.
- **Cost:** whatever hosting n8n costs.
- **Downside:** one more service to run, monitor, and keep credentials in.

### B. Scheduled agent polling the Vapi API

Instead of listening, a scheduled job wakes on a cadence, asks Vapi's API for
calls completed since it last ran, reads their structured outputs, and writes
them onward. A scheduled cloud agent can do this — it runs on a cron without a
terminal open.

- **Latency:** the poll interval.
- **Cost:** no webhook host to maintain.
- **Downside:** it is still a service — just hosted somewhere else — and if a run
  fails silently, records queue up unnoticed until someone checks.

**For this system, polling latency is genuinely fine.** The caller was handed to
a live human during the call. Nothing downstream is time-critical: the record
exists to log the lead and create a follow-up task. A 15-minute delay changes
nothing for anyone.

That makes B a legitimate way to drop n8n — but on the strength of "a scheduled
job polls the API," not "Claude does it in chat." The distinction matters,
because only one of those runs on Saturday night.

## Two things that constrain either option

**1. Structured outputs have to be turned on first.** Both designs read
`call.artifact.structuredOutputs`. That is not wired up yet — see
`config/vapi-settings.md`. Until it is, either option starts by re-parsing an
English transcript, which is lossy and avoidable.

**2. HIPAA mode is in direct tension with option B.** If HIPAA mode gets enabled,
Vapi **does not store structured outputs by default** — so a job that polls for
them later may find nothing to read. A webhook receiver gets the payload at
call-end regardless. If the PHI question in `docs/compliance.md` comes back
"yes," re-check this before committing to polling.

And either way, whatever receives the record joins the BAA question. Medication
lists don't stop being medication lists because they moved to a different
service.

## Where this stands

Not built. Phase 5 on the roadmap, and it depends on structured outputs landing
first. The option isn't locked in — see `docs/open-questions.md`.

Whichever way it goes, the first workflow is the same:

1. Receive or fetch the end-of-call record.
2. Read the structured intake fields.
3. Put a readable summary somewhere the assigned agent will actually see it.
4. Create the follow-up task — especially for calls where the transfer failed and
   the caller was promised a callback.

## Production delivery controls — required

The receiver is not production-ready until it has all of these:

- HTTPS plus Vapi/Twilio request authentication; reject invalid signatures or
  shared-secret checks before parsing sensitive payloads.
- Fast acknowledgement and asynchronous processing.
- Idempotency using `call.id + event type + output/event id`, backed by a unique
  constraint. Store receipt status separately from processing status.
- Retry with bounded backoff, a dead-letter queue, and an alert on any promised
  callback that has no task.
- Out-of-order event handling using event timestamps/sequence, plus periodic
  reconciliation against the Vapi API.
- PHI-safe logs: no raw transcript, medication, phone, or health fields in
  application/error logs.
- Durable polling cursor with an overlap window if polling is retained.

`flows/intake-schema.json` is AI-extracted content only.
`flows/call-event-envelope.schema.json` supplies authoritative call IDs,
timestamps, runtime node state, assistant/tool versions, and transfer events.
Human acceptance—not `assistant-forwarded-call`—is the success condition.

Step 4 is the one that matters most. It is the other half of the promise the
fallback message makes.
