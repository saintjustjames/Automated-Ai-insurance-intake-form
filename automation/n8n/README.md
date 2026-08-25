# n8n — Post-call automation

Nothing built yet. Phase 5 on the roadmap.

Vapi and n8n are **not** alternatives to each other:
- **Vapi** runs the live call and the script, in real time.
- **n8n** runs after the call ends — logging, notifications, record updates,
  eventually the IntegrityCONNECT lookup.

The webhook hangs off the summary step in the flow, right before transfer.

## Intended first workflow
1. Receive the Vapi end-of-call payload.
2. Parse the structured intake fields.
3. Write a call summary somewhere the agent will actually see it.
4. Create a follow-up task or calendar entry for the assigned agent.
