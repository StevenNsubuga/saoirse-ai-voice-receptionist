# Build log & lessons learned

Real problems hit while building this, and how each was fixed. Most are the kind of thing documentation doesn't warn you about.

## ElevenLabs

- **Webhook tool "Edit as JSON" silently fails to save.** Programmatically pasted JSON never synced into the form's state, so "Add tool" did nothing. *Fix:* build tools in Form mode with real input.
- **"Property description cannot be empty" with nothing visibly empty.** Both the body-level description *and* every property's description are mandatory, and a stray empty property hid between filled ones. *Fix:* inspect the DOM for the empty property and delete it.
- **Duplicating an agent produced blank agents.** "New agent" ≠ "Duplicate agent"; only the latter copies prompt, voice, tools and KB.
- **Post-call webhook configured in the wrong slot.** "Conversation Initiation Client Data Webhook" fires at call *start* and expects a different response; the post-call transcript belongs in the separate Post-Call Webhook section.
- **Inflection and pronunciation.** Raising Stability to 60% fixed choppy sentence-to-sentence pitch. The Irish name "Saoirse" was mispronounced; the UI only accepts dictionary *files*, so I created a pronunciation dictionary through the API (`add-from-rules`, alias `Saoirse` → `Seersha`) and attached it to the agent.
- **LLMs are bad at date arithmetic.** "Next Wednesday" is sometimes resolved wrongly. Root cause: the prompt asks the model to convert phrases to ISO dates. Planned fix: resolve dates deterministically in n8n.

## Twilio

- **401 on number import.** I had pasted a Phone Number SID (`PN…`) where the Account SID (`AC…`) belongs.
- **Trial limits.** Trial accounts can only call/text verified numbers, and a US number may be restricted to domestic SMS, so testing used a verified caller ID.

## n8n

- **Active/Inactive toggle is gone in n8n 2.0.** It's now Publish/Unpublish. Unpublished workflows don't respond to schedule triggers but can still be run manually, ideal for testing a daily scheduled workflow.
- **Empty `{}` item from the calendar node.** The "tomorrow" window was computed from a different day than I assumed, so the query matched nothing and n8n emitted an empty placeholder item. The workflow was behaving correctly; my date assumption was wrong.
- **Field-name mismatch.** The SMS node referenced `$json.phone`/`name`/`time` while the parser emitted `to_number`/`patient_name`/`appointment_time`, so Twilio rejected the "To" number.
- **HMAC verification, step by step.** Needed the *raw* body (webhook Raw Body option), correct header parsing (`t=…,v0=…`), signed payload `timestamp.rawBody`, and "Run Once for Each Item" mode (`$input.item` doesn't exist in all-items mode). A temporary debug node that returned only the signature fields accidentally replaced the real payload downstream, a good reminder that debug code in a pipeline has side effects.
- **Operator precedence bug.** `a || b ? 'x' : 'y'` groups as `(a || b) ? 'x' : 'y'`, which mislabeled nearly every call as "outbound". *Fix:* `a || (b ? 'x' : 'y')`.

## Product / business notes

- Dental practices don't run on Google Calendar; market leaders (Adit, Weave/TrueLark, Arini, LuMay and others) all sell PMS integration as the core feature. The calendar here is a stand-in that proves the voice and automation logic.
- No Irish legal requirement for PI/cyber insurance for a SaaS vendor, but a practice's own indemnity doesn't cover a vendor's errors. Incorporating as a limited company is the first liability lever.
