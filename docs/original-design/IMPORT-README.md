# n8n Import Guide — 6 Workflows

## 1. Import each workflow

In n8n: **Workflows → Create Workflow** (opens blank canvas) → top-right **⋯ menu → Import from File…** → pick the JSON. Repeat for all six:

`w1a-check-availability.json` · `w1b-book-appointment.json` · `w1c-log-callback.json` · `w1d-capture-prsi.json` · `w2-post-call.json` · `w3-outbound-reminders.json`

## 2. Create 2 new credentials (one time)

You already have Google Calendar + Google Sheets. Add:

1. **Header Auth** (search "Header Auth") — Name field: `X-Auth-Token`, Value: `YOUR_SHARED_SECRET`. Call the credential "Riverside Webhook Secret".
2. **Gmail OAuth2** — same Google sign-in as before (needed for the email nodes).
3. *(Later, for W3 only)* **Header Auth** — Name: `xi-api-key`, Value: your ElevenLabs API key. Call it "ElevenLabs Key".

## 3. Attach credentials in each imported workflow

Nodes with a ⚠ red mark need a credential selected from the dropdown at the top of the node panel:

| Node type | Credential |
|---|---|
| Webhook (W1a–W1d) | Riverside Webhook Secret |
| Get Events / Create Event / Tomorrow's Events | your Google Calendar |
| Append … (all Sheets nodes) | your Google Sheets |
| Email … (all Gmail nodes) | your Gmail |
| Call Patient (W3) | ElevenLabs Key |

## 4. Replace placeholders

- **RIVERSIDE_CALENDAR_ID** (calendar nodes in W1a, W1b, W3): easiest fix — in the node, switch the Calendar field from "By ID" to the list mode and pick *Riverside Dental* from the dropdown. (Manual alternative: Google Calendar → settings → Integrate calendar → Calendar ID.)
- **PASTE_LOGS_SPREADSHEET_URL** (Sheets nodes in W1b, W1c, W2): paste the URL of "Riverside Dental — Logs".
- **PASTE_PRSI_SPREADSHEET_URL** (W1d): paste the URL of "Riverside Dental — PRSI Checks". If your tab isn't called `Sheet1`, fix the Sheet name in the node.
- **PRACTICE_EMAIL** (Gmail nodes in W1c, W1d, W2): your own email for the demo.
- **OUTBOUND_AGENT_ID / AGENT_PHONE_NUMBER_ID** (W3 "Call Patient"): leave as-is for now — these don't exist until the ElevenLabs agents are built (Phase 2/3). **Keep W3 inactive until then.**

## 5. Add header rows to the sheets (needed for auto-mapping)

Row 1 of each tab, exactly these column names:

- **Bookings:** `timestamp | name | phone | date | time | treatment | new_patient | ref`
- **Callbacks:** `timestamp | name | phone | reason | category | urgency | status`
- **Call Log:** `timestamp | direction | caller | duration_secs | outcome | summary | transcript`
- **Reminders:** *(unused for now — W3 logging happens in the ElevenLabs dashboard call history)*
- **PRSI Checks / Sheet1:** `timestamp | name | dob | ppsn | phone | status`

## 6. Activate + collect URLs

Toggle **Active** (top right) on W1a, W1b, W1c, W1d, W2 — not W3 yet. Production URLs become:

```
https://saoirse.app.n8n.cloud/webhook/check-availability
https://saoirse.app.n8n.cloud/webhook/book-appointment
https://saoirse.app.n8n.cloud/webhook/log-callback
https://saoirse.app.n8n.cloud/webhook/capture-prsi-details
https://saoirse.app.n8n.cloud/webhook/post-call
```

These go into the ElevenLabs tool configs in Phase 2.

## 7. Quick test (terminal or ask Claude)

```bash
curl -X POST https://saoirse.app.n8n.cloud/webhook/check-availability \
  -H "Content-Type: application/json" \
  -H "X-Auth-Token: YOUR_SHARED_SECRET" \
  -d '{"preferred_date":"2026-07-21","preferred_time_of_day":"morning","treatment":"checkup"}'
```

Expect JSON with up to 3 slots. Add a busy event to the calendar and re-run — that slot should vanish.

## Known simplifications (fine for demo, revisit before go-live)

- Irish Summer Time (`+01:00`) is hardcoded in W1a/W1b — swap to `+00:00` late Oct–late Mar (or ask Claude to make it automatic).
- W2 has no HMAC signature verification yet — add before real client data.
- W2 payload field paths may need one adjustment after your first real call (marked with a NOTE inside the Code node).
- W3 reminder outcomes are visible in the ElevenLabs call history rather than a Reminders tab.
