# PASTE THIS ENTIRE PROMPT INTO CLAUDE CODE (inside your DentalBookingAppointment project folder)

You are setting up a new project called **DentalBookingAppointment**. This project builds a dental clinic appointment booking system entirely as n8n workflows on my self-hosted n8n instance. Complete ALL of the following setup tasks in order, then confirm each one is done.

## TASK 1 — Install the n8n MCP

Install and configure the n8n MCP from https://github.com/czlonkowski/n8n-mcp so you can search nodes, validate configurations, and create/update workflows directly in my n8n instance.

Configure it in a project-scoped `.mcp.json` file with these connection details:
- N8N_API_URL: https://n8n.jessiepongaron.cloud/
- N8N_API_KEY: [PASTE-YOUR-N8N-API-KEY-HERE]

Then create a `.gitignore` that excludes `.mcp.json` and any `.env` files — credentials must never be committed.

After configuring, verify the MCP connection works by listing my existing workflows, and tell me what you find.

## TASK 2 — Install the n8n skills

Install the n8n skills from https://github.com/czlonkowski/n8n-skills into this project so they are available in every session. Confirm which skills were installed.

## TASK 3 — Create the CLAUDE.md file

Create a CLAUDE.md in the project root containing everything below, organized clearly. This file is your permanent instruction set for all future sessions in this project.

### Project purpose
A zero-cost, two-channel dental booking system (web + Facebook Messenger) built as n8n workflows. Google Calendar is the availability source, Google Sheets is the booking ledger, Gmail sends confirmations. No SMS. You (Claude Code) build all workflows via the n8n MCP based on my prompts — always use MCP tools to look up node schemas and validate workflows before deploying; never guess node parameters from memory.

### Existing resources (use these, do not recreate)
- Google Drive folder / spreadsheet "JessieDentalCare": https://drive.google.com/drive/folders/1kFOD9-xQc2RgCK_sUjRn8BNupcOVWfdv
  — booking ledger columns: booking_id | timestamp | channel | name | contact | service | date | time | calendar_event_id | status
- Existing n8n credential: "Gmail OAuth2 API - JessieDentalCare" (reference by name in Gmail nodes)
- Google Calendar and Google Sheets credentials: check the n8n instance for existing ones; if missing, ask me to create them before building nodes that need them
- Meta App: JessieDentalCare (currently in Development Mode — only app admins/testers get bot replies until App Review)
- Facebook Page: https://www.facebook.com/jessiedentalcare (Messenger link: m.me/jessiedentalcare)
- Page Access Token and Webhook Verify Token: ask me when needed; store as n8n credentials or env vars, never hardcode in workflow JSON
- n8n instance: https://n8n.jessiepongaron.cloud/ (self-hosted, Docker on Hostinger VPS)

### Business rules (booking logic)
- Clinic hours: 9:00 AM – 6:00 PM, EVERY day, timezone Asia/Manila (+08:00)
- Slot duration: 60 minutes → 9 slots/day (starts 9AM–5PM; every slot must END by 6PM)
- A day is FULLY BOOKED when all 9 slots are taken → system must offer the next available days: scan up to 14 days ahead, suggest up to 3 open days with remaining slot counts
- If requested time is taken but the day still has space → offer the remaining free slots that same day
- No past bookings; same-day bookings need at least 2 hours lead time
- Double-booking guard: re-check Google Calendar for conflicts immediately before creating any event

### Architecture (3 workflows, prefix all names with [DBA])
1. [DBA] Booking Engine — Execute Sub-workflow Trigger (inputs: name, contact, service, date, time, channel) → Google Calendar Get Events for a 14-day range → Code node availability logic → on success: Create Calendar event + append ledger row. Returns one of three statuses:
   - {"status":"available","booking_id":"...","start":"ISO","end":"ISO"}
   - {"status":"time_taken","requested_date":"YYYY-MM-DD","same_day_alternatives":["10:00 AM",...]}
   - {"status":"day_full","requested_date":"YYYY-MM-DD","alternative_days":[{"date":"...","slots_left":5,"free_slots":[...]}]}
2. [DBA] Web Booking — Webhook POST /webhook/dental-booking → normalize input → call Booking Engine → Respond to Webhook + Gmail confirmation. Plus companion endpoints: GET available-slots (per date) and GET month-availability (per month: returns {"YYYY-MM-DD": free_slot_count} so the booking page can grey out full days)
3. [DBA] Messenger Booking — Webhook for Meta callbacks: handle GET verification handshake (echo hub.challenge when hub.verify_token matches) and POST message/postback events → conversation state in a Google Sheets tab (sender_id | current_step | collected_fields | last_updated) → quick-reply-driven flow (no free-text AI parsing in v1) → call Booking Engine → reply via Messenger Send API (HTTP Request node)

Both channel workflows must handle all three Booking Engine statuses. Booking logic lives ONLY in the Booking Engine — never duplicate it in channel workflows.

### Working rules
- Validate every workflow with MCP validation tools before creating or updating it in my instance
- Prefer native n8n nodes; HTTP Request only where no native node exists (e.g., Messenger Send API)
- After deploying/updating a workflow, report its name, ID, and webhook URL(s)
- NEVER activate a workflow that touches the live Google Calendar without asking me first
- PH Data Privacy Act: collect only name, contact, service, schedule; consent notice on the web form; store nothing beyond the ledger columns

### Build order
1. [DBA] Booking Engine (test standalone with pinned data)
2. [DBA] Web Booking + availability endpoints
3. Static booking webpage (HTML/JS consuming the availability endpoints + booking webhook)
4. [DBA] Messenger Booking (dev-mode testing, then App Review)
5. Later: daily reminders, cancel/reschedule flow, Groq free-text parsing

## TASK 4 — Report back

When Tasks 1–3 are complete, give me:
1. Confirmation the MCP is connected (with the list of my existing workflows)
2. The list of installed skills
3. The final CLAUDE.md contents for my review
4. Any missing prerequisites you found (e.g., missing Google Calendar/Sheets credentials in n8n)

Do NOT build any workflows yet — setup only. Wait for my go signal before building [DBA] Booking Engine.
