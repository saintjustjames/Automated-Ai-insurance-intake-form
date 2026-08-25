# CLAUDE.md — Project Context

> Read this first. It is the single source of truth for how this project works and
> what the current state is. If you change something material, update this file
> and `docs/decisions.md` in the same commit.

## What this is

**Prep and Seal Insurance** — an AI voice intake agent that talks to inbound
Medicare/Medicaid callers *while they are on hold*, gathers their details, and
hands them to a licensed human agent who is already up to speed.

The pitch: instead of hold music, the caller spends that time getting prepped.
The licensed agent picks the plan and closes. The AI never does.

## The hard rule

**The AI gathers facts. That is all.**

It never recommends a plan, names a carrier, quotes a price, determines
eligibility, gives medical advice, or completes an enrollment. Every enrollment
is finalized by a licensed human agent. This is a compliance boundary, not a
style preference — do not soften it in any prompt edit.

## Stack

| Layer | Tool | Status |
|---|---|---|
| Phone line | Twilio | Connected. Currently using Vapi's built-in number instead of the Twilio trial number. |
| Conversation | Vapi | Live. Single assistant, full script loaded. |
| Live handoff | `transfer_to_licensed_agent` (Vapi tool) | Configured as warm transfer w/ voicemail fallback. **Untested end to end.** |
| Post-call automation | n8n | Not built. Planned. |
| Carrier/member data | Ritter → IntegrityCONNECT | Not built. Needs real dev work. |

Account email for all services: `prepandsealinsurance@gmail.com`

## Where things live

```
prompts/system-prompt.md    The live Vapi prompt. PASTE PAYLOAD ONLY - see below.
prompts/README.md           The paste rule. Read before editing the prompt.
prompts/CHANGELOG.md        Why the prompt changed, version by version.
tests/call-scenarios.md     Manual call tests. P0 block gates any go-live.
flows/branching-logic.md    The branch map. THE SPEC the prompt implements.
flows/intake-flow.json      Machine-readable mirror of the branch map.
flows/intake-schema.json    The record one call produces. Contract for n8n.
tools/*.json                Intended Vapi tool config, version-controlled.
config/vapi-settings.md     Non-prompt Vapi config: voice, transcriber, tools.
docs/decisions.md           Locked decisions. Don't relitigate these.
docs/open-questions.md      Unresolved. Answer these before building past them.
docs/roadmap.md             Build phases in order.
docs/compliance.md          CMS/SOA rules that constrain the script.
automation/n8n/             Future post-call workflows.
```

## Working agreement

- **`prompts/system-prompt.md` is the payload, not a document about it.** Select
  all, copy, paste into Vapi. Never put a version header, a repo path, a dev
  note, markdown formatting, or a `[bracket]` placeholder in it — the model reads
  every one of those as instructions. Notes for humans go in
  `prompts/README.md`, `prompts/CHANGELOG.md`, or `flows/branching-logic.md`.
  There is no automated deploy, so say plainly when it needs re-pasting.
- **If the agent says it's doing something, it must call the tool in that same
  turn.** Never write a prompt line that announces an action without the tool
  call attached. That failure — narrating a transfer that never happens — is the
  original bug this project started from.
- **Change the branch map first, then the prompt.** `flows/branching-logic.md` is
  the spec; the prompt is one implementation of it. Editing the prompt alone
  makes the two drift, and the branch map is the only place the *whole* flow —
  interrupts included — is visible at once.
- Never add a capability to the prompt that isn't actually wired up in Vapi. The
  agent must not claim to save, look up, call back, or enroll. It *can* transfer.
- Keep questions one at a time. Callers are often seniors on hold.
- When a decision gets made in conversation, write it into `docs/decisions.md`.
  Chat history is not the record. This repo is.

## Current state, in one line

English single-assistant intake is built and live in Vapi with a real transfer
tool. It has **not** been tested end to end on a live call. That test is the next
thing that matters.
