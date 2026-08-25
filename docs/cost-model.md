# Cost Model

What a call actually costs, and which levers are worth pulling — in order of
return per unit of risk.

Everything here is an estimate. Replace it with real numbers from your own Vapi
call logs once volume exists; that data beats any blog post.

---

## What you're paying for

A Vapi call bills in four layers, not one:

| Layer | Rough cost | Notes |
|---|---|---|
| Vapi orchestration | **$0.05/min**, fixed | The advertised price. It is the smallest piece. |
| LLM | ~$0.002–$0.10/min | Depends entirely on model and how many turns the call runs. |
| TTS (voice) | varies | Often rivals the LLM. A premium multilingual voice is not cheap. |
| STT (transcriber) + telephony | varies | Twilio minutes plus the transcriber. |

Published all-in figures for real setups land around **$0.10–$0.30/min**, with
cheap stacks near $0.15 and premium stacks $0.35–$0.40.

A full intake runs roughly **4–6 minutes**. Call it **$0.60–$1.50 per completed
intake** before tuning.

---

## The thing that dominates LLM cost on a voice call

**The system prompt is re-sent on every single turn.**

This intake has ~15 questions, and a real call takes more turns than that once
you count acknowledgments, spell-backs, and repair. Call it ~30 model turns.

The prompt is roughly 2,400 tokens. So:

```
2,400 tokens  ×  ~30 turns  ≈  72,000 input tokens per call
```

Almost all of that is the *same text* over and over. That is the cost, and it is
also the lever.

---

## Lever 1 — Prompt caching. Do this first.

**Free. No quality risk. Largest single saving.**

Vapi passes the model call through to the provider, so prompt caching works
whenever the underlying provider supports it — Anthropic, OpenAI 4o-class, Groq.
Anthropic's prefix caching is documented at roughly **90% off cached input
tokens** and materially lower latency on long prompts.

On the ~72,000 input tokens above, nearly all of it is cacheable prefix.

Two rules make or break it:

1. **The prefix must stay byte-stable.** Any change anywhere in the prefix
   invalidates everything after it.
2. **Nothing dynamic near the top.** Interpolating a timestamp, call ID, or
   caller name into the top of the system prompt defeats caching for every call.

**We are currently in good shape on both.** `prompts/system-prompt.md` is fully
static — no variables, no interpolation. That was a side effect of writing it as
a clean paste payload, and it happens to be exactly what caching wants.

**Keep it that way.** If you ever want the caller's name or ZIP inside the system
prompt, put it at the *end*, after all the static instructions — never near the
top. The time-of-day greeting belongs in `firstMessage`, which is separate from
the cached system prompt and does not affect this.

Verify it's actually working by checking cached-token counts in the call logs. If
they're zero across repeated calls, something is silently invalidating the
prefix.

---

## Lever 2 — Right-size the model. Do this second, and test it.

Cheaper models are a real saving, but this is the lever with actual downside, so
it goes after caching, not before.

Current Anthropic pricing per million tokens:

| Model | Input | Output |
|---|---|---|
| Claude Opus 5 | $5.00 | $25.00 |
| Claude Sonnet 5 | $3.00 ($2.00 promotional through 2026-08-31) | $15.00 ($10.00) |
| Claude Haiku 4.5 | $1.00 | $5.00 |

Rough input cost for one call at ~72,000 tokens, before caching:

| Model | Uncached | With caching |
|---|---|---|
| Opus 5 | ~$0.36 | ~$0.04 |
| Sonnet 5 | ~$0.22 | ~$0.02 |
| Haiku 4.5 | ~$0.07 | ~$0.01 |

Note what that table says: **caching a mid-tier model beats downgrading without
caching, by a wide margin.** Doing both is best, but the order matters — pull the
free lever before the risky one.

### The risk, stated plainly

This is not a chatty assistant where a weaker model just sounds blander. This
agent has to:

- call `transfer_to_licensed_agent` **in the same turn** it announces the
  transfer — the exact failure this project already hit once
- take exactly one of five branches at the program-status question
- never invent a medication name from a garbled one
- never name a carrier, quote a price, or judge eligibility
- never leak internal status vocabulary into speech

Those are instruction-following behaviors, and instruction-following is what
degrades first on a smaller model. A cheaper model that occasionally narrates a
transfer without making one costs far more than it saves.

**So don't guess — measure.** `tests/call-scenarios.md` exists for exactly this.
Any model change re-runs the **P0 block**, plus scenarios 3, 4, 19, and 27, which
target the specific behaviors a downgrade would break first.

That turns the model choice from a gamble into an experiment: try Haiku, run the
suite, and keep it only if it passes clean. If it doesn't, you've learned
something concrete for the price of an hour of test calls.

---

## Lever 3 — Bring your own API key

Vapi still charges its $0.05/min orchestration fee either way, but with your own
provider key the token costs go to your provider account at native pricing rather
than through Vapi's margin. Worth doing once volume is real.

---

## Lever 4 — Keep the call short

Every turn re-sends the prompt, so turn count is a cost driver, not just a UX
one. Things already in the prompt that help:

- one or two sentences per turn
- the summary read-back capped at essentials, ~15 seconds
- correcting only the item the caller flagged, not re-reading the whole summary
- an immediate transfer when someone asks for a human — which is the *cheapest*
  possible call as well as the right one

Also set a **max call duration** (see `config/vapi-settings.md`). A wedged call
that loops for forty minutes is a billing problem as much as a quality one.

---

## What not to cut

**The voice.** A warmer preset and slightly slower speed are doing real work for
an audience of seniors on hold. Downgrading the voice to save a fraction of a
cent per minute trades the thing callers actually experience for the smallest
line item on the bill.

**The three separate medication questions.** Merging them back into one saves a
couple of turns and reliably produces worse data from a 74-year-old reading a
pill bottle. That data is the product.

---

## Sources

- Vapi pricing breakdowns (2026): [Zeeg](https://zeeg.me/en/blog/post/vapi-ai-pricing), [pxlpeak](https://pxlpeak.com/blog/ai-tools/vapi-pricing-breakdown), [layer3labs](https://www.layer3labs.io/guides/vapi-pricing)
- Prompt caching behavior and the byte-stable-prefix rule: [Future AGI on Vapi latency](https://futureagi.com/blog/how-to-optimize-vapi-latency-2026/), [PromptHub](https://www.prompthub.us/blog/prompt-caching-with-openai-anthropic-and-google-models)
- Anthropic model pricing: current as of August 2026; re-check at
  https://docs.claude.com/en/docs/about-claude/pricing before budgeting.
