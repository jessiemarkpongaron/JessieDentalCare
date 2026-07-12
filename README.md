# JessieDentalCare — Dental Booking System

A zero-cost, two-channel dental appointment booking system (web + Facebook Messenger) built as [n8n](https://n8n.io) workflows on a self-hosted instance.

- **Google Calendar** — availability source of truth
- **Google Sheets** — booking ledger
- **Gmail** — booking confirmations

## Booking page

The static booking page lives in [`webpage/index.html`](webpage/index.html) — a single self-contained HTML file (no frameworks, no build step). It consumes the live n8n endpoints:

| Endpoint | Method | Purpose |
|---|---|---|
| `/webhook/dental-booking` | POST | Create a booking (name, contact, service, date, time) |
| `/webhook/available-slots?date=YYYY-MM-DD` | GET | Free slots for a date |
| `/webhook/month-availability?month=YYYY-MM` | GET | Free-slot counts per day (greys out full days) |

Base URL: `https://n8n.jessiepongaron.cloud/webhook`

## Business rules

- Clinic hours 9:00 AM – 6:00 PM daily, Asia/Manila time
- 60-minute slots → 9 slots/day (last slot starts 5:00 PM)
- No past bookings; same-day bookings need 2+ hours lead time
- Bookings accepted up to 14 days ahead
- Double-booking guard: calendar re-checked immediately before event creation

## Architecture

Three n8n workflows, all prefixed `[DBA]`:

1. **`[DBA] Booking Engine`** — all availability/slot logic; called as a sub-workflow by both channels
2. **`[DBA] Web Booking`** — the webhook endpoints above + Gmail confirmation
3. **`[DBA] Messenger Booking`** — Facebook Messenger quick-reply flow (planned)

## Privacy

Per the PH Data Privacy Act (RA 10173), only name, contact, service, and schedule are collected, with an explicit consent notice on the booking form.
