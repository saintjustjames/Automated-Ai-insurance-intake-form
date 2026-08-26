# Multi-assistant ("sub-agents") — analysis

**Short answer:** worth doing, mostly for reliability and language quality, not
for cost. The cost saving is real but smaller than it looks, and partly cancels
itself out. **Not yet** — it's Phase 4 for good reasons, and one of them is that
splitting a system that has never completed a single end-to-end call means
debugging a distributed system that never worked as a simple one.

---

## What "sub-agents" means in Vapi

Two mechanisms, and the guidance has changed:

| | What it is | Status |
|---|---|---|
| **Squads** | Multiple assistants that hand off to each other mid-call, each owning part of the conversation. Per-assistant model, voice, and transcriber. | **The recommended path.** |
| **Workflows** | Visual node graph with explicit edges and per-node config. | **No longer recommended for new builds.** Vapi found current models can't reliably hold "the current node's instructions plus all possible next steps," and reports Squads produce better results. |

So if this happens, it's Squads with handoffs — not the node-graph Workflow
you'd otherwise reach for on a fixed 15-question intake. Worth knowing before
building the wrong thing.

---

## The cost argument, honestly

### What splitting actually saves

The system prompt is re-sent every turn — roughly 2,400 tokens × ~30 turns ≈
72,000 input tokens per call. Split into five assistants, each turn only carries
its own assistant's prompt:

| Assistant | Prompt | Turns | Input tokens |
|---|---|---|---|
| Greeter / language | ~250 | 3 | 750 |
| Contact capture | ~450 | 7 | 3,150 |
| Coverage + qualifiers | ~700 | 9 | 6,300 |
| Clinical | ~550 | 7 | 3,850 |
| Close + transfer | ~450 | 4 | 1,800 |
| | | | **~15,850** |

On paper that's a ~78% cut in input tokens.

### Why it's smaller than that in practice

**Splitting fragments the prompt cache.** Each assistant has its own prefix, so
each one pays its own cache write, and the reuse is spread across five smaller
prefixes instead of concentrated in one big one. The two levers are working on
the same tokens — you don't get both savings stacked.

And the reuse that matters here is *within a single call*: one prefix, ~30 turns,
all inside five minutes. That's where caching earns its keep, and it's exactly
the thing splitting breaks up. A single 2,400-token prompt cached across 30 turns
is already close to the cheapest version of this call.

**Neither changes the fixed costs.** Vapi's $0.05/min orchestration, TTS,
transcription, and Twilio minutes are untouched by how many assistants you use.
Those are a large share of the per-minute bill.

### The one real cost lever in here

**Per-assistant model selection.** This is the part worth actually wanting.

Most of this call is mechanical: capture a name, read a phone number back, take
a ZIP. That does not need a frontier model. A few parts carry real judgment:

- the five-way program-status branch
- the medication spell-back loop, where inventing a drug name is the worst
  failure in the system
- the close, where the transfer tool has to fire in the same turn it's announced

Running a cheap model for capture and a capable one only where judgment lives is
a better cost/quality trade than downgrading the whole call to the cheapest model
that survives the test suite. **That is the argument for splitting that actually
holds up on cost** — not token count.

---

## The reliability argument, which is stronger

A 2,400-token prompt asks the model to simultaneously hold 15 questions, 12
interrupt conditions, the guardrails, the tool discipline, and the branching.
Instruction-following degrades as that load grows, and it degrades first on
exactly the behaviors this project has already had to fix once.

Five focused prompts are each easier to follow than one long one. If a cheaper
model can't pass the P0 suite as a monolith, it may well pass as a specialist
handling seven turns of contact capture.

**And language.** This is the strongest case of all, and it's already the plan:
a Spanish-native script with a Spanish voice beats one assistant switching
languages mid-call, and Creole — where transcription is the open risk — can be
tuned or pulled independently without touching English. That's Phase 2–4, and
it's the natural moment to introduce Squads.

---

## What splitting would break if done carelessly

Three specific hazards for *this* agent. The first two are safety and compliance
issues, not quality issues.

**1. Global interrupts must live in every assistant.** The emergency stop, the
immediate transfer on "let me talk to a person," and the frustration escalation
are not steps in the flow — they fire at any point. If the clinical assistant
doesn't carry them, a caller describing chest pain during the medication question
gets asked which pharmacy they use. That is the failure mode to design against
first.

**2. Consent state must survive every handoff.** Two flags in particular:

- `plan_naming_suppressed` — set when the caller declines Scope of Appointment
  consent. If it doesn't carry across a handoff, a later assistant names plan
  types to someone who explicitly declined. **That's a compliance failure**, not
  a cosmetic one.
- `is_proxy` — if it's lost, the medication list gets attributed to the caller
  instead of the person they're calling for.

**3. The transfer tool must be attached to every assistant**, not just the
closer. Otherwise the immediate-transfer interrupt silently can't fire from four
fifths of the call.

Each of these turns one prompt-level rule into five copies that must stay in
sync. `flows/branching-logic.md` is the spec that keeps them consistent — this is
what that file is for.

---

## Recommendation

**Not now.** In order:

1. **No end-to-end call has ever succeeded** (open question #7). Splitting first
   means debugging handoffs in a system whose basic path is unproven.
2. **Compliance items gate go-live anyway** — PHI/BAA, recording consent, carrier
   approval of the SOA script. None of them get easier with five assistants.
3. **The test suite has to exist first.** Splitting multiplies the surface: every
   handoff is a new place for state to drop. Without P0 passing on the monolith,
   there's no baseline to compare against.

**Do it when Spanish lands.** That's Phase 2, and it's where the payoff is real
and the cost argument stops being the point. Introducing Squads for language is
also the lowest-risk way to learn the handoff mechanics — a language router is
one decision at the top of the call, not five mid-call state transfers.

If cost is the pressing concern before then, the cheaper moves are: confirm
whether Vapi is actually setting cache breakpoints at all, cap max call duration,
and test a cheaper model against the P0 suite. All three are reversible in a
dashboard. A Squad rewrite is not.

---

## Proposed split, for when it happens

Each assistant carries the global interrupts, the transfer tool, and the consent
flags. Only the questions differ.

| # | Assistant | Nodes | Model tier |
|---|---|---|---|
| A | Greeter + language router | G0, GI-12 | Cheapest that can route reliably |
| B | Contact capture | N2–N5 | Cheap — mechanical, digit read-back |
| C | Coverage + qualifiers | N6–N11 | Capable — the five-way branch and consent gate |
| D | Clinical | N12 | Capable — highest hallucination stakes |
| E | Close + transfer | N13–N16 | Capable — tool discipline, summary accuracy |

Test implication: every handoff boundary needs its own scenario proving the
consent flags, proxy flag, and collected record survive it — plus one emergency
and one "let me talk to a person" fired from *each* assistant, not just the first.

---

## Sources

- [Assistants vs Squads vs Workflows](https://vapi.ai/community/m/1414880041480749056) — current guidance
- [Workflows overview](https://docs.vapi.ai/workflows/overview) · [Squads](https://docs.vapi.ai/squads)
- [Vapi review 2026 (Coval)](https://www.coval.ai/blog/vapi-review-2026-is-this-voice-ai-platform-right-for-your-project/) — notes Workflows no longer recommended for new builds
