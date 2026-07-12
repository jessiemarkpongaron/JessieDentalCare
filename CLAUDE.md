# DentalBookingAppointment — Project Instructions

## Project Purpose

A **zero-cost, two-channel dental booking system** (web + Facebook Messenger) built entirely as n8n workflows on a self-hosted n8n instance.

- **Google Calendar** = availability source of truth
- **Google Sheets** = booking ledger
- **Gmail** = booking confirmations
- **No SMS.**

Always use n8n MCP tools to look up node schemas and validate workflows before deploying — **never guess node parameters from memory**.

## Existing Resources (use these — do NOT recreate)

- **Google Drive folder "JessieDentalCare"**: https://drive.google.com/drive/folders/1kFOD9-xQc2RgCK_sUjRn8BNupcOVWfdv
  - Ledger columns: `booking_id | timestamp | channel | name | contact | service | date | time | calendar_event_id | status`
- **Existing n8n credential**: `Gmail OAuth2 API - JessieDentalCare` (reference by name in Gmail nodes)
- **Google Calendar credential**: `Google Calendar OAuth2 - JessieDentalCare` (id: `JyhCdC7NHQFoSXnZ`, type: `googleCalendarOAuth2Api`)
- **Clinic Google Calendar ID**: `1d2753be1d85374806f98dd5edf65900092e87e60d14a78237377baa3b49ec06@group.calendar.google.com` — use this calendar for all availability checks and event creation
- **Google Sheets credential**: `Google Sheets account` (id: `5nQuSuKm6fH8qEra`, type: `googleSheetsOAuth2Api`)
- **Ledger spreadsheet**: "JessieDentalCare Booking" (id: `1m8rdun3DBGwzltC4dV9-fZSoAZl3YGq9egdh2JC-TeA`) — tab `booking_ledger` for bookings, tab `messenger_booking` for Messenger conversation state
- **Deployed workflow**: `[DBA] Booking Engine` (id: `5DmOXVRV4Ff7wbPm`, ACTIVE) — call via Execute Sub-workflow node; also returns a defensive 4th status `{"status":"invalid","message":"..."}` for malformed input, and enforces a 14-day max booking window (matches the 14-day calendar fetch). Extra optional input `mode` (book|slots|month): read-only query modes used by the availability endpoints so all slot math lives in the engine — `mode=slots` + date → `{status:"slots",date,free_slots[],slots_left}`; `mode=month` + date=YYYY-MM → `{status:"month",month,days:{"YYYY-MM-DD":count}}`
- **Deployed workflow**: `[DBA] Web Booking` (id: `hceCdjZbCri0IX7r`, ACTIVE) — live endpoints (CORS `*`):
  - POST https://n8n.jessiepongaron.cloud/webhook/dental-booking (body: name, contact, service, date, time; 400 on invalid; Gmail confirmation sent when contact is an email)
  - GET https://n8n.jessiepongaron.cloud/webhook/available-slots?date=YYYY-MM-DD
  - GET https://n8n.jessiepongaron.cloud/webhook/month-availability?month=YYYY-MM
- **Meta App**: JessieDentalCare (Development Mode — only app admins/testers get replies until App Review passes)
- **Facebook Page**: https://www.facebook.com/jessiedentalcare (Messenger: m.me/jessiedentalcare)
- **Page Access Token and Webhook Verify Token**: ask the user when needed; store as n8n credentials/env vars — **never hardcode in workflow JSON**
- **n8n instance**: https://n8n.jessiepongaron.cloud/ (self-hosted, Docker, Hostinger VPS)

## Business Rules

- Clinic hours **9:00 AM – 6:00 PM every day**, timezone **Asia/Manila (+08:00)**
- **60-minute slots → 9 slots/day** (slot starts 9 AM–5 PM; every slot must END by 6 PM)
- **Day FULLY BOOKED** = all 9 slots taken → offer next available days: scan up to **14 days ahead**, suggest up to **3 open days** with remaining slot counts
- **Requested time taken but day has space** → offer remaining free slots that same day
- **No past bookings**; same-day bookings need **2+ hours lead time**
- **Double-booking guard**: re-check Google Calendar for conflicts immediately before creating any event

## Architecture (3 workflows, all names prefixed `[DBA]`)

### 1. `[DBA] Booking Engine`
- Execute Sub-workflow Trigger — inputs: `name, contact, service, date, time, channel`
- → Google Calendar **Get Events** for the 14-day range
- → **Code node** availability logic
- → on success: **Create Calendar event** + **append ledger row**
- Returns **exactly one** of:
  ```json
  {"status":"available","booking_id":"...","start":"ISO","end":"ISO"}
  {"status":"time_taken","requested_date":"YYYY-MM-DD","same_day_alternatives":["10:00 AM","..."]}
  {"status":"day_full","requested_date":"YYYY-MM-DD","alternative_days":[{"date":"...","slots_left":5,"free_slots":["..."]}]}
  ```

### 2. `[DBA] Web Booking`
- Webhook **POST** `/webhook/dental-booking` → normalize input → call Booking Engine → Respond to Webhook + Gmail confirmation
- Companion endpoints:
  - **GET** `available-slots` (per date)
  - **GET** `month-availability` (per month: returns `{"YYYY-MM-DD": free_slot_count}` so the booking page can grey out full days)

### 3. `[DBA] Messenger Booking`
- Webhook for Meta callbacks:
  - Handle **GET verification handshake** (echo `hub.challenge` when `hub.verify_token` matches)
  - Handle **POST** message/postback events
- Conversation state in a Google Sheets tab: `sender_id | current_step | collected_fields | last_updated`
- **Quick-reply-driven flow** (no free-text AI in v1)
- → call Booking Engine → reply via **Messenger Send API** (HTTP Request node)

**Both channels must handle all three Booking Engine statuses. Booking logic lives ONLY in the Booking Engine — never duplicate it in channel workflows.**

## Working Rules

- **Validate every workflow with MCP validation tools** before creating/updating it
- Prefer **native n8n nodes**; HTTP Request only where no native node exists (Messenger Send API)
- After deploying/updating, report the **workflow name, ID, and webhook URL(s)**
- **NEVER activate a workflow that touches the live Google Calendar without asking the user first**
- **PH Data Privacy Act**: collect only name, contact, service, schedule; consent notice on the web form; store nothing beyond the ledger columns

## Build Order

1. `[DBA] Booking Engine` (test standalone with pinned data)
2. `[DBA] Web Booking` + availability endpoints
3. Static booking webpage (HTML/JS consuming the endpoints)
4. `[DBA] Messenger Booking`
5. Later: reminders, cancel/reschedule, Groq free-text parsing
