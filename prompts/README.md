# prompts/

## The paste rule

`system-prompt.md` is **the payload, not a document about the payload.** Select
all, copy, paste into the Vapi assistant's system prompt field. Nothing in it is
addressed to a human.

That means it deliberately contains **no**:

- version headers, status lines, or "paste this into Vapi" notes
- references to files in this repo — the model can't open `docs/open-questions.md`
  and shouldn't be told it exists
- markdown emphasis, bullets, backticks, or blockquotes — the prompt tells the
  agent never to speak formatting, so putting formatting *in* the lines it speaks
  is working against itself
- square-bracket placeholders like `[name]` or `[ZIP]` — a model under load will
  eventually say "bracket name" out loud. Substitution is described in prose
  instead
- internal field names or status vocabulary in any line meant to be spoken

Everything a human needs to know *about* the prompt lives in
`../flows/branching-logic.md` (what the flow is and why), `CHANGELOG.md` (what
changed and why), and `../config/vapi-settings.md` (the settings around it).

## Why this split exists

The earlier version of this file was both at once — a markdown document with a
version header, dev asides, and repo paths, that was also being pasted into Vapi
verbatim. The model was reading instructions written for a developer, including
pointers to files it cannot see. `## # IDENTITY` was going into the context
window as literal text.

Splitting them costs one extra file and removes an entire category of bug.

## Editing order

1. Change `../flows/branching-logic.md` first. That's the spec.
2. Change `system-prompt.md` to match.
3. Log why in `CHANGELOG.md`.
4. Re-paste into Vapi by hand. There is no automated deploy.
