# AI Voice Secretary (n8n + Vapi.ai)

An n8n workflow that turns a text message in Slack into an actual phone call to a manager, transcribes their spoken reply, routes it back to the right person, and logs it as a task in Odoo — a voice-first "secretary" that sits between a team's chat and a manager's phone.

📹 **[Demo video](./demo/demo.mp4)** — a real end-to-end run of the workflow.

> Business context is anonymized (placeholder names, phone numbers, and channel IDs) — this is a portfolio export of a working automation, not the live production config.

## How it works

1. **Slack trigger** — a message lands in a Slack channel (e.g. a question that needs the manager's decision).
2. **Outbound call via Vapi.ai** — the workflow places a real phone call to the manager, using a voice AI agent that opens with a summary of the Slack message and waits for the manager to actually respond out loud, rather than firing a fixed script and hanging up.
3. **Poll until the call ends**, then pull the transcript from Vapi's call status endpoint.
4. **Branch on outcome:**
   - **No answer / voicemail** → falls back to a synthesized voice message summarizing what was asked, so nothing gets silently dropped.
   - **Manager responded** → the reply gets logged into Odoo as a task (via a custom Odoo model), and a confirmation is posted back to Slack.
5. **Topic-based routing** — a parsing step splits the manager's spoken reply into segments by topic (e.g. a payroll question vs. a parts/procurement question that both got asked in the same original Slack message) and routes each segment to the right Slack channel/person, rather than dumping one raw transcript on everyone.

## Why this is a harder problem than it looks

- The call has to **wait for the person to actually start talking** (`firstMessageMode: assistant-waits-for-user`) rather than talking over them or timing out too early — tuned against real silence/duration thresholds.
- A single spoken reply often answers **multiple unrelated questions** from the original message ("tell person A this, and person B that") — the transcript-splitting logic detects topic boundaries inside continuous speech rather than assuming one topic per call.
- The **failure path is a first-class case**, not an afterthought: a missed call still produces a useful fallback voice message instead of the workflow just silently failing.

## Stack

- n8n (workflow orchestration, Code nodes in JavaScript)
- Vapi.ai (outbound voice AI calling + transcription)
- Slack API (trigger + message routing)
- Odoo (custom model for task logging)

## Setup

Import `ai-secretary-workflow.json` into n8n, then:

1. Configure Slack and Odoo credentials inside n8n (referenced by the workflow, not stored in the export).
2. Set these environment variables for the Vapi.ai integration:
   ```
   VAPI_API_KEY=...
   VAPI_ASSISTANT_ID=...
   VAPI_PHONE_NUMBER_ID=...
   SECRETARY_TARGET_PHONE=+1XXXXXXXXXX
   ```
3. Update the Slack channel IDs in the relevant nodes to your own workspace's channels.
