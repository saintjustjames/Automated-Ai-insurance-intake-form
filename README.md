# Prep and Seal — AI Intake Agent

AI voice agent that talks to inbound Medicare/Medicaid callers while they're on
hold, gathers their details, and hands them to a licensed human agent who's
already up to speed.

The AI gathers facts. A licensed human closes. Always.

## Start here

- **[CLAUDE.md](CLAUDE.md)** — full project context. Claude Code reads this
  automatically.
- **[flows/branching-logic.md](flows/branching-logic.md)** — the branch map.
  Every node, every branch, every interrupt. This is the spec.
- **[prompts/system-prompt.md](prompts/system-prompt.md)** — the live Vapi
  prompt. This is the product.
- **[docs/open-questions.md](docs/open-questions.md)** — what's unresolved.
- **[docs/roadmap.md](docs/roadmap.md)** — what's next, in order.

## Layout

```
CLAUDE.md                   Project context for Claude Code
README.md                   You are here
prompts/
  system-prompt.md          The live Vapi prompt — paste payload, nothing else
  README.md                 The paste rule; read before editing the prompt
  CHANGELOG.md              Why the prompt changed, version by version
tests/
  call-scenarios.md         Manual call tests; the P0 block gates go-live
flows/
  branching-logic.md        The branch map — the spec the prompt implements
  intake-flow.json          Machine-readable mirror of the branch map
  intake-schema.json        The record one call produces (contract for n8n)
tools/
  transfer_to_licensed_agent.json   Intended Vapi transfer tool config
  end_intake_call.json              Intended Vapi end-call tool config
config/
  vapi-settings.md          Voice, transcriber, tools — the non-prompt config
docs/
  decisions.md              Settled. Don't relitigate.
  open-questions.md         Unresolved. Answer before building past them.
  roadmap.md                Build phases, in order
  compliance.md             CMS/SOA constraints on the script
  cost-model.md             What a call costs; which levers to pull, in order
  multi-assistant.md        Squads: worth it for reliability and language, not cost
automation/
  README.md                 Post-call automation options (not built yet)
```

## Deploying a prompt change

There's no automated deploy.

1. Change `flows/branching-logic.md` first — that's the spec.
2. Change `prompts/system-prompt.md` to match. It is a **paste payload**: no
   headers, no repo paths, no markdown, no `[brackets]`. See
   [prompts/README.md](prompts/README.md).
3. Select all, paste into the Vapi assistant's system prompt field by hand.
4. Log why in `prompts/CHANGELOG.md`.
5. Re-run the **P0 block** in [tests/call-scenarios.md](tests/call-scenarios.md).
   Prompt edits are not local — tightening one instruction routinely loosens
   another elsewhere.

## Status

English single-assistant intake is live in Vapi with a real transfer tool.
It has not been tested end to end on a live call. That's the next thing.
