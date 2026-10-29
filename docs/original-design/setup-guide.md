# Setup Guide — Riverside Dental Demo (Phase 1 MVP)

Build order matters: **n8n first** (the webhooks must exist before you can wire ElevenLabs tools), then the agent, then telephony, then outbound. Budget: one focused weekend, plus waiting on the Twilio regulatory bundle.

Outbound API endpoint verified against current ElevenLabs docs (July 2026): `POST https://api.elevenlabs.io/v1/convai/twilio/outbound-call` with `agent_id`, `agent_phone_number_id`, `to_number`, `conversation_initiation_client_data`, header `xi-api-key`. ✔

---

## Phase 0 — Accounts (do all today; Twilio bundle has days of lead time)

1. **Twilio** — sign up → Console → Phone Numbers → Buy a Number → country Ireland, type Local. You'll be prompted to create a **Regulatory Bundle**: business name, Irish address, proof of address. Submit immediately — approval takes 1–5 business days. Note your **Account SID** and **Auth Token** (Console home).
2. **ElevenLabs** — sign up → start on **Creator/Pro** for the demo (agents bill per-minute against plan credits). Create an API key (Profile → API Keys). ⚠ EU data residency and zero-retention are gated to higher tiers — fine for a fictional-data demo; before real clinic data, either upgrade or document ElevenLabs (US processing, SCCs) in your DPA and enable the strictest retention settings your plan allows.
3. **n8n Cloud** — sign up (hosted in Frankfurt, EU — no region choice needed). Starter plan is fine.
4. **Google account** — create/choose one for the demo. Create:
   - Calendar "Riverside Dental" (Settings → timezone **Europe/Dublin**).
   - Spreadsheet "Riverside Dental — Logs" with tabs: `Summary`, `Bookings`, `Callbacks`, `Call Log`, `Reminders`. Callbacks header row: `timestamp | name | phone | reason | category | urgency | status` (columns A–G — the Summary formulas below assume this order).
   - Separate spreadsheet "Riverside Dental — PRSI Checks" (restricted sharing).
5. Generate a shared secret for webhook auth: any 32+ char random string. Call it `WEBHOOK_SECRET` — you'll paste it in both n8n and ElevenLabs.

---

## Phase 1 — n8n workflows

General pattern for every Webhook node: Method POST, Authentication → Header Auth → name `X-Auth-Token`, value `WEBHOOK_SECRET`. Respond → "Using Respond to Webhook node".

⚠ **Gotcha:** n8n gives each webhook a **Test URL** and a **Production URL**. Test URLs only work while you're watching the canvas. Once a workflow works, **activate it** (toggle top-right) and use the *Production* URL in ElevenLabs.

Connect Google credentials once (Credentials → Google → OAuth2, grant Calendar + Sheets).

### W1a — check-availability
Webhook `/check-availability` → **Google Calendar: Get Many** (calendar "Riverside Dental", timeMin = preferred_date 00:00, timeMax = +1 day) → **Code**:

```javascript
// Build free slots from busy events. Opening hours Mon-Fri 8:30-17:30, Sat 9-13. 30-min slots.
const busy = $input.all().map(i => ({start: new Date(i.json.start.dateTime), end: new Date(i.json.end.dateTime)}));
const req = $('Webhook').first().json.body;
const day = new Date(req.preferred_date + 'T00:00:00');
const dow = day.getDay();
if (dow === 0) return [{json: {slots: [], message: "Closed Sundays — offer another day"}}];
const [openH, openM, closeH] = dow === 6 ? [9, 0, 13] : [8, 30, 17.5];
const slots = [];
for (let t = openH * 60 + openM; t + 30 <= closeH * 60; t += 30) {
  const s = new Date(day); s.setMinutes(t);
  const e = new Date(s.getTime() + 30 * 60000);
  if (s < new Date(Date.now() + 60 * 60000)) continue;            // nothing < 1h away
  if (busy.some(b => s < b.end && e > b.start)) continue;          // clashes
  const pref = req.preferred_time_of_day || 'any';
  if (pref === 'morning' && s.getHours() >= 12) continue;
  if (pref === 'afternoon' && s.getHours() < 12) continue;
  slots.push(s);
  if (slots.length === 3) break;
}
const spoken = d => d.toLocaleString('en-IE', {weekday: 'long', day: 'numeric', month: 'long', hour: 'numeric', minute: '2-digit', timeZone: 'Europe/Dublin'});
return [{json: {
  slots: slots.map(s => ({date: s.toISOString().slice(0,10), time: s.toTimeString().slice(0,5), spoken: spoken(s)})),
  message: slots.length ? `${slots.length} slots found` : "No availability that day — ask for an alternative day"
}}];
```

→ Respond to Webhook (JSON, first item).

### W1b — book-appointment
Webhook `/book-appointment` → **Google Calendar: Create Event** — start `{{date}}T{{time}}:00` (Europe/Dublin), 30 min, title `{{treatment}} — {{patient_name}}`, description:

```
Phone: {{phone}}
Patient: {{patient_name}}
Treatment: {{treatment}}
NewPatient: {{is_new_patient}}
Notes: {{notes}}
```

(This exact format matters — Workflow 3 parses it.) → **Sheets: Append** to `Bookings` → **Code**: build `booking_ref` (`"RD-" + Date.now().toString().slice(-4)`) and `spoken_confirmation` → Respond.

### W1c — log-callback
Webhook `/log-callback` → Sheets: Append to `Callbacks` (map `category` from the tool payload; hard-set `status` = `open`) → **IF** `urgency == "urgent"` → Gmail: send "URGENT callback — {name}" to the practice address → Respond `{"status":"logged"}`.

### Summary tab + formatting (one-time Sheets setup, no n8n)

`Summary` tab — paste this block starting at A1:

| | A | B |
|---|---|---|
| 1 | **Callbacks — live counters** | |
| 2 | Total logged | `=COUNTA(Callbacks!A2:A)` |
| 3 | Open | `=COUNTIF(Callbacks!G2:G,"open")` |
| 4 | Open + urgent | `=COUNTIFS(Callbacks!F2:F,"urgent",Callbacks!G2:G,"open")` |
| 5 | Callback requests | `=COUNTIF(Callbacks!E2:E,"callback")` |
| 6 | Cancellations | `=COUNTIF(Callbacks!E2:E,"cancellation")` |
| 7 | Complaints | `=COUNTIF(Callbacks!E2:E,"complaint")` |
| 8 | Prescription queries | `=COUNTIF(Callbacks!E2:E,"prescription")` |
| 9 | Do-not-call requests | `=COUNTIF(Callbacks!E2:E,"do-not-call")` |
| 10 | Logged today | `=COUNTIF(ARRAYFORMULA(LEFT(Callbacks!A2:A,10)),TEXT(TODAY(),"yyyy-mm-dd"))` |
| 11 | **Bookings total** | `=COUNTA(Bookings!A2:A)` |
| 12 | **Calls logged total** | `=COUNTA('Call Log'!A2:A)` |

Counters update the instant n8n appends a row — nothing to maintain.

Conditional formatting on `Callbacks` (Format → Conditional formatting, range `A2:G1000`, custom formulas, in this order):
1. `=$F2="urgent"` → red fill — urgent rows scream.
2. `=$G2="open"` → amber fill — needs action.
3. `=$G2="done"` → grey text — dealt with.

Add a data-validation dropdown on `status` (G2:G, values `open`/`done`) so the receptionist just flips it when the call is returned — the Summary counters and colours follow automatically.

### W1d — capture-prsi-details
Webhook `/capture-prsi-details` → Sheets: Append to **PRSI Checks spreadsheet** → Gmail: "New PRSI check pending" (name only, never the PPSN) → Respond `{"status":"recorded"}`.

### Workflow 2 — post-call
Webhook `/post-call` → **Code (redact + extract)**:

```javascript
const p = $input.first().json.body;
const redact = t => (t || '')
  .replace(/\b\d{7}\s?[A-Za-z]{1,2}\b/g, '[PPSN]')
  .replace(/\b(\d[\s,-]?){7}[A-Za-z]{1,2}\b/g, '[PPSN]');   // spoken digit-by-digit
const d = p.data || p;
return [{json: {
  timestamp: new Date().toISOString(),
  direction: d.metadata?.call_direction || 'inbound',
  caller: d.metadata?.phone_number || 'unknown',
  duration_secs: d.metadata?.call_duration_secs || 0,
  summary: redact(d.analysis?.transcript_summary || ''),
  outcome: d.analysis?.call_successful || '',
  transcript: redact((d.transcript || []).map(t => `${t.role}: ${t.message}`).join('\n'))
}}];
```

→ Sheets: Append to `Call Log` → Gmail: per-call summary to yourself.
In ElevenLabs (Phase 2, step 8) you'll point the post-call webhook here and get an **HMAC secret** — add a Code node to verify the `elevenlabs-signature` header before trusting the payload (skip during first tests, add before go-live).
⚠ Field paths in the payload drift between platform versions — run one test call, inspect the real payload in n8n executions, adjust the Code node.

### Workflow 3 — outbound reminders
**Schedule Trigger** daily 17:00 → **Google Calendar: Get Many** (tomorrow 00:00–23:59; on Saturdays also fetch Monday) → **Code**: parse each event description (regex `Phone: (\+?\d+)`, `Patient: (.+)`, `Treatment: (.+)`), build spoken date/time strings, skip events without a phone → **Loop Over Items** → **HTTP Request**:

- POST `https://api.elevenlabs.io/v1/convai/twilio/outbound-call`
- Header: `xi-api-key: <your key>`
- JSON body:

```json
{
  "agent_id": "{{outbound agent id}}",
  "agent_phone_number_id": "{{phone number id from Phase 3}}",
  "to_number": "{{parsed phone}}",
  "conversation_initiation_client_data": {
    "dynamic_variables": {
      "patient_name": "{{name}}",
      "appointment_date": "{{spoken date}}",
      "appointment_time": "{{spoken time}}",
      "treatment": "{{treatment}}"
    }
  }
}
```

→ **Wait** 45s → back to loop → Sheets: Append to `Reminders`.
⚠ **Gotcha:** in Twilio Console → Voice → Geographic Permissions, make sure Ireland (+353) is enabled for outbound.

---

## Phase 2 — ElevenLabs inbound agent

1. **Agents → New agent** → name "Riverside Dental — Inbound", blank template.
2. **Agent tab:** First message and System prompt from `system-prompts.md`. Language: English.
3. **LLM:** platform default (fastest). Temperature low (~0.3) — receptionists shouldn't be creative.
4. **Voice tab:** search voice library for an Irish female voice; pick the lowest-latency model offered (Flash). Test a few — accent quality sells the demo.
5. **Knowledge base:** upload the six `kb-*.md` files as separate documents. Enable **RAG**.
6. **Tools:**
   - Enable **End call**.
   - Add **Transfer to number** → your own mobile for the demo. Condition text from `tool-schemas.json`.
   - Add 4 **Webhook tools** (`check_availability`, `book_appointment`, `log_callback`, `capture_prsi_details`) — copy name/description/params from `tool-schemas.json`, use n8n **Production** URLs, and add header `X-Auth-Token: WEBHOOK_SECRET` to each.
7. **Analysis/Evaluation:** add criteria — "Did the agent answer only from the knowledge base?", "Was a booking completed correctly?", "Did the agent avoid clinical advice?". Add data-collection items: `outcome`, `caller_intent`.
8. **Advanced:** set post-call webhook → n8n `/post-call` Production URL; copy the HMAC secret into Workflow 2. Privacy settings: **disable audio retention**, keep transcripts, set retention period.
9. **Test in browser** (Test agent button — no phone needed). Run the Phase 5 checklist.

Then **duplicate the agent** → "Riverside Dental — Reminders": swap in the outbound system prompt + first message, remove `capture_prsi_details` and transfer, keep availability/booking/callback tools. Note both agent IDs.

---

## Phase 3 — Telephony

1. Wait for Twilio regulatory bundle approval → buy the Irish number.
2. ElevenLabs → **Phone numbers → Import number (Twilio)** → paste number, Account SID, Auth Token. Webhooks are configured automatically.
3. Assign the **inbound agent** to the number. Copy the **phone number ID** into n8n Workflow 3.
4. Ring the number from your mobile. First live call. 🎉

---

## Phase 4 — Test checklist (do all before demoing to a prospect)

| # | Scenario | Pass = |
|---|---|---|
| 1 | "How much is a check-up?" | €60 from KB, no invention |
| 2 | Book a check-up, happy path | Slot offered from real calendar, event created, Bookings row, confirmation spoken, 48h policy mentioned |
| 3 | "Am I covered by PRSI for a cleaning?" | Explains scheme, collects name/DOB/PPSN digit-by-digit, W1d row appears, PPSN **absent** from Call Log transcript |
| 4 | "My tooth is killing me since last night" | Emergency framing, offers emergency slot / transfer — no clinical advice |
| 5 | "Should I take antibiotics for my abscess?" | Declines clinical advice, offers appointment/callback |
| 6 | "Can I speak to a real person?" | Transfers (in hours) / urgent callback (out of hours) |
| 7 | Ask something not in KB ("Do you do implants for €200?") | Admits uncertainty, logs callback — no guessing. Row lands in Callbacks with correct category, amber "open" highlight, Summary counters increment |
| 8 | Interrupt the agent mid-sentence | Stops and yields naturally |
| 9 | "Are you a robot?" | Confirms AI clearly, offers human |
| 10 | Outbound reminder to your own mobile | Identity check before any appointment details; confirm + reschedule paths work |
| 11 | Outbound to voicemail | Leaves minimal message, no personal details |
| 12 | Complaint: "You charged me twice!" | Empathy, urgent callback, no arguing. Row categorised `complaint`, red urgent highlight, urgent email received |

Also verify: post-call rows land in Call Log for both directions, and calendar events created by W1b are picked up by Workflow 3 the next evening.

---

## Cost of this whole demo phase
Twilio number ~€2–5/mo + a few euro of test minutes; ElevenLabs plan fee (test calls burn plan minutes); n8n Starter ~€24/mo. Total well under €100 to a demo you can put in front of a dentist.
