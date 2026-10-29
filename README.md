# Saoirse — AI Voice Receptionist for Irish Dental Practices

A working prototype of a 24/7 AI phone receptionist for Irish SMBs (dental first), built end-to-end with **ElevenLabs Agents**, **Twilio** and **n8n**, with Claude as my pair-programmer throughout. It answers calls in an Irish voice, answers practice questions from a knowledge base, books real appointments, logs callbacks, captures PRSI details safely, and texts appointment reminders.

> **Status:** functional pilot/demo against a fictional practice ("Riverside Dental"). Not yet deployed to a real clinic. Everything here is sanitized: no real keys, numbers, sheet IDs or emails.

## How it works

```
Caller ──► Twilio number ──► ElevenLabs agent "Saoirse"
                               │  (Claude Haiku 4.5 · Irish voice · 6-doc knowledge base)
                               │
        4 webhook tools (header-authenticated) ──► n8n
          ├─ check_availability ──► W1a ──► Google Calendar search
          ├─ book_appointment  ──► W1b ──► Calendar event + booking log row
          ├─ log_callback      ──► W1c ──► Sheet row + urgent email to staff
          └─ capture_prsi_details ► W1d ──► Restricted sheet, PPSN redacted, staff email

Call ends ──► ElevenLabs post-call webhook (HMAC-signed)
          ──► W2: verify signature ► redact PII ► log call ► email summary

Daily 17:00 ──► W3: tomorrow's calendar events ► parse ► Twilio SMS reminder
```

## Repo layout

| Path | What's in it |
|---|---|
| `n8n-workflows/` | Six importable n8n workflow exports (W1a–W1d tools, W2 post-call logging, W3 SMS reminders) |
| `elevenlabs/` | Agent system prompts, webhook tool schemas, the 6 knowledge-base documents |
| `docs/` | Capability diagram, [build log & lessons learned](docs/build-log.md), original design docs |

## Stack

- **ElevenLabs Agents** — voice, LLM orchestration (Claude Haiku 4.5 for phone-call latency), knowledge base, tool calling, post-call analysis
- **Twilio** — telephony and SMS
- **n8n** — workflow automation connecting the agent to Google Calendar, Sheets and Gmail

## Design decisions worth noting

- **Tools, not trust.** The LLM never touches the calendar directly; it calls narrow, authenticated webhooks. Each webhook requires a shared-secret header.
- **Signed post-call webhook.** W2 verifies ElevenLabs' HMAC-SHA256 signature (with timestamp replay protection) before processing anything.
- **Privacy by design.** PPSNs are redacted in transcripts and summaries; PRSI details go to a restricted sheet and never into email; audio isn't stored; conversation retention is capped at 30 days.
- **SMS over outbound calls for reminders.** I originally built outbound voice reminders, then swapped to SMS: less intrusive, cheaper, and patients prefer it.
- **No clinical advice.** The agent is explicitly bounded to admin tasks, with urgent/emergency routing to staff.

## Known limitations / roadmap

- Google Calendar stands in for a practice management system. Real deployments need PMS integration (Dentally has a public API; Exact sits behind Henry Schein One's API Exchange).
- Relative dates ("next Wednesday") are currently resolved by the LLM and occasionally wrong. Planned fix: pass the phrase through and resolve it deterministically in an n8n Code node.
- Twilio is on a trial account; production needs an Irish number and paid account.

## Setting it up yourself

1. Create the credentials listed in `.env.example` (n8n credentials / ElevenLabs settings).
2. Import each file in `n8n-workflows/` into n8n and replace the `YOUR_...` placeholders (sheet IDs, calendar ID, staff email, Twilio number, HMAC secret).
3. In ElevenLabs, create the agent using `elevenlabs/system-prompts.md`, add the knowledge-base docs, and create the four webhook tools from `elevenlabs/tool-schemas.json`, pointing at your n8n production webhook URLs.
4. Set the post-call webhook to W2 with HMAC auth.

`docs/original-design/` holds the initial setup guide. It predates two later changes: W3 switched from voice calls to SMS, and W2 gained HMAC verification.

## License

MIT
