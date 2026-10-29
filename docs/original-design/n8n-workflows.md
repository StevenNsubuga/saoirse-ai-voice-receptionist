# n8n Workflows — MVP (3 workflows)

All on n8n Cloud, **EU-hosted region** (choose Frankfurt when creating the account). Every webhook node: add a required header `X-Auth-Token: <your-shared-secret>` and set the same header in the matching ElevenLabs tool.

---

## Workflow 1: Booking tools (serves check_availability + book_appointment)

**W1a — check-availability**

1. **Webhook** (POST `/webhook/check-availability`) — receives `preferred_date`, `preferred_time_of_day`, `treatment`.
2. **Google Calendar → Get Availability** (or Cal.com `GET /v2/slots` via HTTP Request) — query the practice calendar for free slots on/near the date, within opening hours (Mon–Fri 8:30–17:30, Sat 9–13).
3. **Code node** — pick up to 3 slots matching time-of-day preference; build the `spoken` string for each ("Tuesday the 21st at half nine").
4. **Respond to Webhook** — return `{ slots: [...], message }`. If none: `{ slots: [], message: "no availability, offer alternative days" }`.

**W1b — book-appointment**

1. **Webhook** (POST `/webhook/book-appointment`).
2. **Google Calendar → Create Event** — title: `{{treatment}} — {{patient_name}}`, description includes phone + new-patient flag + notes. If `is_reschedule`, add "RESCHEDULED — check for old event" to description (staff resolve manually at MVP; automate later).
3. **Google Sheets → Append Row** — bookings log: timestamp, name, phone, date, time, treatment, new patient, booking_ref.
4. **Respond to Webhook** — `{ status: "confirmed", booking_ref, spoken_confirmation }`.

**W1c — log-callback**

1. **Webhook** (POST `/webhook/log-callback`).
2. **Google Sheets → Append Row** — callbacks tab: timestamp, name, phone, reason, **category** (from the tool), urgency, status = `open`.
3. **IF urgency = urgent** → **Gmail/SMTP node** — immediate email to practice (subject: "URGENT callback — {name}").
4. **Respond to Webhook** — `{ status: "logged" }`.

**W1d — capture-prsi-details** (PPSN — handle with extra care)

1. **Webhook** (POST `/webhook/capture-prsi-details`).
2. **Google Sheets → Append Row** — to a **separate spreadsheet** ("PRSI Checks"), owned by the clinic, shared only with staff who verify entitlements — NOT the general log sheet: timestamp, name, DOB, PPSN, phone, status (pending/verified/failed).
3. **Gmail/SMTP** — notify staff: "New PRSI entitlement check pending" (name only in the email — never put the PPSN in an email body).
4. **Respond to Webhook** — `{ status: "recorded", message: "Details recorded, staff will confirm entitlement before the appointment" }`.
5. Staff verify in the DSP Treatment Benefit portal as they do today, and mark the row verified. Delete rows once verified + appointment complete (retention: 3 months max).

---

## Workflow 2: Post-call processing

1. **Webhook** (POST `/webhook/post-call`) — set this URL as the agent's post-call webhook in ElevenLabs. Payload includes transcript, call summary, duration, evaluation results, collected data. **Verify the HMAC signature header** ElevenLabs sends (shared secret from the webhook settings page).
2. **Code node — PPSN redaction** — regex-mask PPS numbers in the transcript and summary before anything is stored: `/\b\d{7}[A-Za-z]{1,2}\b/g → "[PPSN]"` (also catch digit-by-digit spoken forms, e.g. 7+ digits separated by spaces followed by a letter). The raw PPSN lives ONLY in the restricted PRSI sheet via W1d — never in the call log.
3. **Code node** — extract: caller number, duration, summary, outcome (answered / booked / callback / transferred), evaluation scores.
4. **Google Sheets → Append Row** — call log tab (this sheet is your pilot-metrics dataset: containment rate, volumes, outcomes).
5. **Gmail/SMTP** — send the owner a short per-call summary email (MVP) — switch to a daily 6pm digest (Schedule Trigger + Sheets read) once volume grows.

---

## Workflow 3: Outbound reminders

1. **Schedule Trigger** — daily 17:00 Mon–Sat.
2. **Google Calendar → Get Events** — tomorrow's appointments (and Monday's on Saturday runs).
3. **Code node** — for each event, parse patient name, phone (from event description), date, time, treatment. Skip events without a phone number.
4. **Loop → HTTP Request** — `POST https://api.elevenlabs.io/v1/convai/twilio/outbound-call` with header `xi-api-key`, body:
   ```json
   {
     "agent_id": "<outbound-agent-id>",
     "agent_phone_number_id": "<imported-twilio-number-id>",
     "to_number": "+3538XXXXXXXX",
     "conversation_initiation_client_data": {
       "dynamic_variables": {
         "patient_name": "Mary Kelly",
         "appointment_date": "Tuesday the 21st of July",
         "appointment_time": "half nine in the morning",
         "treatment": "check-up"
       }
     }
   }
   ```
   *(Verify exact endpoint path and field names against current ElevenLabs docs before wiring — API shape occasionally changes.)*
5. **Wait node** — 30–60s between calls (don't ring the whole book at once).
6. **Google Sheets → Append Row** — reminders log: who was called, when, API response.
7. Call outcomes (confirmed / rescheduled / cancelled) arrive automatically through **Workflow 2** (post-call webhook fires for outbound calls too) and reschedules go through **W1a/W1b**.

---

## Sheets structure (one spreadsheet, four tabs)

| Tab | Columns |
|---|---|
| Bookings | timestamp, name, phone, date, time, treatment, new_patient, ref |
| Callbacks | timestamp, name, phone, reason, category, urgency, status (open/done) |
| Summary | live COUNTIF dashboard over the other tabs — no n8n writes here (see setup guide for formulas) |
| Call Log | timestamp, direction, caller, duration, outcome, summary, eval_score |
| Reminders | timestamp, name, phone, appt_date, api_status |

Plus the separate restricted **PRSI Checks** spreadsheet (W1d): timestamp, name, dob, ppsn, phone, status.

**GDPR note:** keep spreadsheets in the client's own Google account where possible (you access it as processor), set a retention rule (delete rows > 12 months; PRSI rows > 3 months), and document Sheets + n8n as sub-processors in your client DPA.

**PPSN legal note:** PPSN use is restricted under Irish law (Social Welfare Consolidation Act 2005) to specified bodies and their agents. Dental practices registered with the Department of Social Protection for the Treatment Benefit Scheme collect PPSNs lawfully for that purpose — your system captures it only in that flow, as the practice's processor. Record this purpose limitation in each client DPA, and cover phone collection + AI transcription of PPSNs in the DPIA for healthcare clients. Also disable audio retention in ElevenLabs for clinics (transcript-only), so no PPSN sits in stored audio.
