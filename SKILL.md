---
name: artist-calendar
description: |
  Build a self-hosted scheduling page for an artist (or any individual host)
  to replace Calendly. Visitors book time slots through a request-then-approve
  flow; the host gets one-click approve/reject emails; approved bookings auto-
  create Google Calendar events with unique Meet links. Owned data, on-brand,
  no third-party iframe. Triggers: "artist calendar", "custom calendly",
  "scheduling page", "booking framework", "private booking system",
  "self-hosted scheduler", "open windows" (the original).
author: Dave Krugman + Claude
version: 1.1.0
date: 2026-11-05
---

# Artist Calendar

A blueprint for building a self-hosted scheduling system that replaces Calendly.
This skill describes the **structure** — not the styling. The user brings their
own brand; the architecture stays the same.

## When to use

Trigger when the user wants their own booking page hosted on their own domain,
with their own design. Common phrasings:

- "Build me a custom Calendly"
- "I want a booking page on my site"
- "Scheduling system that's on-brand"
- "I'm canceling Calendly"
- "Artist calendar"

Don't use for:

- Embedding Calendly via widget (use Calendly's own docs)
- Multi-host enterprise scheduling with shared inboxes
- Anything requiring complex calendar federation across orgs

## How this skill works

This is a build runbook, not a self-executing skill. When triggered:

1. Ask the user the questions in **Initial questions** below
2. Walk them through the architecture in **What you're building**
3. Implement in the order described in **Build order**
4. Surface the gotchas before they bite (see **Lessons from production**)

The user supplies their own styling — colors, fonts, animations, copy. The
file structure, API contracts, KV schema, OAuth flow, email flow, and resilient
patterns are the reusable part.

---

## Initial questions

Before writing any code, get answers via AskUserQuestion:

1. **Stack**
   - Next.js App Router (recommended; this skill assumes it)
   - Other React framework — adapt accordingly
   - Plain server-rendered HTML — significant rework; suggest porting later

2. **Storage**
   - Vercel KV (recommended — atomic ops, integrated with Vercel)
   - Upstash Redis (same API as Vercel KV, works anywhere)
   - Other key-value store — note the slot-claim race needs `SET NX` semantics

3. **Email provider**
   - Resend (recommended — clean SDK, great DX, easy domain verification)
   - SendGrid / Mailgun / Postmark — swap the SDK; logic identical

4. **Host's Google account type**
   - Google Workspace — unique Meet link per booking via Calendar API ✅
   - Consumer Gmail — `conferenceData.createRequest` does NOT work reliably;
     use a static permanent Meet URL via env var as fallback
   - No Google — substitute Zoom (paid + their API) or skip auto-conferencing
     and have the host send the link manually in a reply email

5. **Event types for v1**
   - Single (recommended — ship faster)
   - Multiple — architecture supports it via type slug in keys/routes; defer

6. **Approval model**
   - Request → host approves → confirmation (recommended — matches Calendly's
     "request" feature, lets host screen visitors)
   - Auto-confirm on submit — simpler; lose screening

7. **Visitor experience tradeoffs**
   - "Unverified app" warning if host stays in OAuth Testing mode — fine for
     host but visitors don't OAuth, so no impact on them
   - 7-day refresh token expiration in Testing mode — recommend publishing the
     app even without verification (warning is harmless)

---

## What you're building

A three-role system on the user's own domain:

| Role | Surface | Path |
|---|---|---|
| Visitor | Public booking page | `/<booking-slug>` (e.g. `/open-windows`, `/book`, `/sessions`) |
| Host | Email with approve/reject buttons; result page after click | inbox → `/<booking-slug>/decision` |
| Admin (= host) | OAuth setup + clear-all dashboard | `/dashboard/booking-setup` (or similar) |

**Conflict-aware out of the box.** The booking page reads the host's actual
Google Calendar (via the `freebusy.query` API) before offering slots. Any
existing event — meetings, personal events, vacation, focus blocks — is
treated as unavailable. The host blocks time by *creating a Calendar event
the way they normally would*; the booking page picks it up automatically.
No admin UI for blocking time is needed — the host's existing Calendar
workflow IS the blocker.

### Flow

```
Visitor                                                Host
  │                                                      │
  ▼                                                      │
/<booking-slug>  ← 3-column picker (day → time → form)   │
  │                                                      │
  ▼                                                      │
POST /api/booking                                        │
  │ atomic SET-NX in KV (booking { status: pending })    │
  ▼                                                      │
sendRequestEmails()                                      │
  ├──► Visitor: "Request received"                       │
  └──► Host: "[Action needed]" with                  ────┤
       APPROVE / REJECT buttons (links with token)       │
                                                         ▼
                                            GET /api/booking/decide
                                              ?slot=...&token=...&action=approve|reject
                                                         │
                                                         ▼
                                              ┌─────────────────────────┐
                                              │  approve                │
                                              │  • patch KV status      │
                                              │  • Google Calendar      │
                                              │    events.insert with   │
                                              │    conferenceData       │
                                              │    → unique Meet link   │
                                              │  • sendConfirmedEmails  │
                                              │    (with .ics)          │
                                              │                         │
                                              │  reject                 │
                                              │  • patch KV status      │
                                              │  • delete Calendar evt  │
                                              │    (if exists)          │
                                              │  • sendRejectedEmail    │
                                              └─────────────────────────┘
                                                         │
                                                         ▼
                                              /<booking-slug>/decision
                                              ?outcome=approved|rejected
```

---

## File structure

Aim for this layout (adjust paths to taste):

```
src/
  lib/
    booking/
      config.ts        # single source of truth for all knobs
      timezone.ts      # DST-correct TZ helpers via Intl.DateTimeFormat
      slots.ts         # slot generator + validator
      storage.ts       # KV CRUD + atomic slot claiming
      ics.ts           # RFC 5545 .ics generator
      email.ts         # Resend wrappers for 3 flows
      google.ts        # Calendar API + OAuth + KV-backed refresh token
  app/(site)/
    <booking-slug>/
      page.tsx
      page.module.css
      decision/
        page.tsx
        page.module.css
    dashboard/booking-setup/
      page.tsx
      page.module.css
    api/
      booking/
        route.ts                  # POST new booking
        slots/route.ts            # GET available slots
        decide/route.ts           # GET approve/reject from email
      admin/
        google-oauth/
          start/route.ts          # POST → returns consent URL
          callback/route.ts       # GET ← Google redirect
          status/route.ts         # GET status, DELETE to disconnect
        bookings/
          clear/route.ts          # POST nuke all bookings
  components/
    BookingPage/                  # public booking UI (3-column flow)
    BookingDashboard/             # admin dashboard
```

---

## Build order

Build in this order so you can test each piece before moving on:

### 1. Dependencies

```bash
npm install resend @vercel/kv googleapis google-auth-library
```

### 2. `src/lib/booking/config.ts`

Define ALL configuration in one file. Don't sprinkle constants across modules.

```typescript
export const HOST_TIMEZONE = 'America/New_York';
export const HOST_NAME = '<the host name>';
export const HOST_EMAIL = process.env.BOOKING_HOST_EMAIL?.trim() || '<default>';

/** Meeting duration in minutes */
export const SLOT_MINUTES = 15;

/**
 * Cadence between slot start times. Must be >= SLOT_MINUTES.
 * Buffer = SLOT_INTERVAL_MINUTES - SLOT_MINUTES.
 * e.g. SLOT_MINUTES=15, SLOT_INTERVAL_MINUTES=25 → 10-min buffer.
 */
export const SLOT_INTERVAL_MINUTES = 25;

export const BOOKING_HORIZON_DAYS = 14;
export const MIN_LEAD_MINUTES = 60;

/**
 * Weekly availability. `[dayOfWeek, startH, startM, endH, endM]`.
 * dayOfWeek per JS Date.getDay() (0=Sun, 6=Sat).
 * Multiple entries per day are allowed (lunch block split, etc).
 */
export const AVAILABILITY: ReadonlyArray<[number, number, number, number, number]> = [
  [1, 9, 30, 17, 0],
  // ...etc
];

export const EVENT = {
  title: '<event title>',
  description: '<event description>',
  get location(): string {
    // Falls back to static Meet URL if Calendar API isn't connected
    return process.env.STATIC_MEET_URL?.trim() || '';
  },
} as const;
```

### 3. `src/lib/booking/timezone.ts`

Zero-dep TZ helpers. The trick: format a UTC Date in the target TZ via
`Intl.DateTimeFormat`, then compute the offset from the rendered wall-clock.
Handles DST correctly because `Intl` is TZ-aware.

Key functions:
- `wallClockToUtc(year, month, day, hour, minute, tz)` → returns UTC Date
- `getWallClock(date, tz)` → returns `{ year, month, day, hour, minute, dayOfWeek }`
- `formatInTz(date, tz, opts)` → human-readable string

### 4. `src/lib/booking/slots.ts`

```typescript
interface Slot {
  id: string;            // ISO UTC of start
  start: Date;
  durationMinutes: number;
}

interface SlotRange { from: Date; to: Date }

export function generateSlots({ from, to }: SlotRange): Slot[];
export function defaultRange(now?: Date): SlotRange;
export function isValidSlotId(id: string, now?: Date): boolean;
```

The slot generator iterates days in the host TZ, finds matching AVAILABILITY
rules (using `.filter` not `.find` to support multi-window days), and emits
slots that:
- Start aligned to the cadence
- Have their END fit within the window (start + duration ≤ dayEnd)

`isValidSlotId` should check membership in `generateSlots(defaultRange())`
output — don't try to re-derive the constraints inline. Cheap (50 slots over
14 days) and correct.

### 5. `src/lib/booking/storage.ts`

Vercel KV schema:

```
booking:slot:<ISO_UTC>     → Booking JSON
booking:index              → sorted set { score: epoch_ms, member: slotId }
booking:rejected:<ISO>:<ts> → audit copy of rejected bookings
```

```typescript
interface Booking {
  slotId: string;
  name: string;
  email: string;
  note?: string;
  visitorTimezone?: string;
  createdAt: string;
  status: 'pending' | 'confirmed' | 'rejected';
  approvalToken: string;   // crypto.randomBytes(24).toString('base64url')
  decidedAt?: string;
  googleEventId?: string;  // populated on approve
  meetLink?: string;
  googleEventLink?: string;
}

// Critical: atomic claim via SET NX
export async function tryClaimSlot(booking: Booking): Promise<boolean> {
  const result = await kv.set(slotKey(booking.slotId), booking, { nx: true });
  if (result !== 'OK') return false;
  await kv.zadd(INDEX_KEY, { score: new Date(booking.slotId).getTime(), member: booking.slotId });
  return true;
}
```

Also: `getBooking`, `patchBooking`, `setBookingStatus`, `listUpcoming`,
`clearAllBookings`.

### 6. `src/lib/booking/ics.ts`

RFC 5545 generator. One VEVENT, UTC times, METHOD:REQUEST. Include
`URL:` field with the Meet link so calendar clients show "Visit URL" button.

### 7. `src/lib/booking/email.ts`

Three email helpers via Resend:

- `sendRequestEmails(booking)` → request-received to visitor; action-needed
  to host with Approve/Reject links
- `sendConfirmedEmails(booking)` → confirmation to both (with .ics attached)
- `sendRejectedEmail(booking)` → polite decline to visitor

Approve/reject link format:
`${origin}/api/booking/decide?slot=${slotId}&token=${approvalToken}&action=approve|reject`

### 8. `src/lib/booking/google.ts`

Calendar API integration. Handles OAuth, refresh-token storage in KV, event
creation with `conferenceDataVersion: 1` for Meet link auto-generation.

Critical functions:
- `buildAuthUrl()` → consent URL with state token in KV (10-min TTL)
- `completeAuth(code, state)` → exchange + store refresh token; decode id_token
  to capture the connected email for the dashboard
- `getAuthorizedClient()` → OAuth2Client primed with refresh token
- `createBookingEvent(booking)` → events.insert with retry logic
- `deleteBookingEvent(eventId)` → for reject after approve
- `getBusyRanges(from, to)` → `calendar.freebusy.query` on primary; returns
  `{start, end}[]`. Cached in KV for 60s (key: `google:freebusy:primary`).
  Returns `[]` only if Calendar isn't connected at all. If the API call
  errors, serve the last cached window if it covers the range, otherwise
  THROW so the slots route offers nothing. Never fail open (see Lesson 12).
- `invalidateFreebusyCache()` → `kv.del(FREEBUSY_CACHE_KEY)`. Called after
  the system creates or deletes a Calendar event itself, so freshly-approved
  bookings show as unavailable on the next slot fetch (not 60s later).

Required OAuth scopes:
```
https://www.googleapis.com/auth/calendar.events
https://www.googleapis.com/auth/calendar.freebusy
openid
email
```

`calendar.events` does NOT cover `freebusy.query` — it needs its own scope
(see Lesson 11). Any time the scope list changes, the host must re-connect
from the admin dashboard so the stored refresh token picks up the new grant.

Sanitize the `requestId` in `conferenceData.createRequest` — strip colons/dots
from the ISO slot timestamp (Google silently no-ops on some non-alphanumeric
identifiers).

### 9. API routes

Follow the route structure listed above. **Critical: AWAIT all email sends
and Calendar API calls before returning the response.** Vercel serverless
terminates the function context when you return, killing fire-and-forget
promises. `.catch()` is not enough.

The `GET /api/booking/slots` route should filter out slots that conflict
with either:

1. Slots already claimed via this system (from KV index)
2. Existing events on the host's calendar (from `getBusyRanges`)

Run those two lookups in parallel (`Promise.all`) and filter the generated
slot set against both. Range intersection: a slot `[slotStart, slotEnd)`
overlaps a busy block `[busyStart, busyEnd)` iff
`slotStart < busyEnd && slotEnd > busyStart`.

If `getBusyRanges` throws (Calendar is connected but the lookup failed),
return an empty slot list with a "try again shortly" message rather than
the unfiltered set. Unknown availability means no availability.

Also call `invalidateFreebusyCache()` after `createBookingEvent` and
`deleteBookingEvent` so the next slot fetch reflects the change
immediately rather than 60s later (the KV TTL).

### 10. UI

Three columns / steps:

1. Date picker (group slots by local-TZ day; show day name + slot count)
2. Time grid (slots for selected day, in visitor's local TZ; auto-detect via
   `Intl.DateTimeFormat().resolvedOptions().timeZone`)
3. Name + email + optional note form, with "Selected" pill showing the chosen
   slot

Success state: show "Request received" with the selected time. Don't
auto-redirect — leave the visitor with a clear confirmation.

### 11. Admin dashboard

Behind a password gate (use whatever auth pattern the site already has — an
`ADMIN_SECRET` env var compared to a posted password is the simplest).

The dashboard surfaces:
- Google Calendar connection status (LIVE / OFF)
- **Connected as** email (decoded from id_token at OAuth completion and stored
  in KV — this is the single most useful diagnostic when troubleshooting)
- Connect / Re-connect / Disconnect buttons
- Clear All Bookings button (danger styling; deletes from KV AND Calendar)

---

## Required env vars (Vercel or equivalent)

| Var | Required? | What for |
|---|---|---|
| `RESEND_API_KEY` (or your provider) | yes | Email delivery |
| `ADMIN_SECRET` | yes | Admin dashboard gate |
| `KV_REST_API_URL`, `KV_REST_API_TOKEN`, etc. | yes (auto via Vercel KV integration) | Booking storage |
| `GOOGLE_CLIENT_ID` | yes | OAuth client |
| `GOOGLE_CLIENT_SECRET` | yes | OAuth client |
| `GOOGLE_REDIRECT_URI` | yes | `https://<your-domain>/api/admin/google-oauth/callback` |
| `BOOKING_HOST_EMAIL` | optional | Where approval emails go |
| `BOOKING_FROM_EMAIL` | optional | Email sender override |
| `STATIC_MEET_URL` | optional | Permanent Meet link fallback |
| `NEXT_PUBLIC_SITE_URL` | optional | Origin override for approve links |

---

## One-time setup the host needs to do

1. **Google Cloud project** at console.cloud.google.com
2. **Enable the Calendar API** at the dedicated library page — separate switch
   from OAuth credentials. (This is the #1 missed step; see Gotchas.)
3. **OAuth consent screen** (Google Auth Platform → Audience): External, fill
   contact info. **Publish the app** to avoid 7-day refresh token expiration
   from Testing mode. The "unverified" warning during the host's own OAuth is
   harmless.
4. **Credentials → OAuth 2.0 Client ID** → Web application with:
   - Authorized redirect URI: `https://<your-domain>/api/admin/google-oauth/callback`
   - Also add `http://localhost:3000/api/admin/google-oauth/callback` for dev
5. If host is on **Google Workspace**:
   - admin.google.com → Security → API Controls → Manage Third-Party App
     Access → Configure new app → enter Client ID → set to **Trusted**.
     Without this, Meet conference creation may silently no-op even though
     OAuth succeeds.
6. **Resend** — verify the sender domain at the DOMAIN level (not single
   sender). All aliases (bookings@, contact@, etc.) work once domain is
   verified.
7. **Vercel env vars** — set everything from the table above.
8. After deploy, visit `/dashboard/booking-setup` → enter admin password →
   click **Connect Google Calendar** → carefully select the right account
   (especially if there are multiple signed in).

---

## Lessons from production (read before debugging)

These are the gotchas that cost real time on the first build. Surface them
early.

### 1. Vercel serverless kills fire-and-forget promises
Every email send and Calendar API call in routes MUST be `await`-ed before
returning. The function context terminates the instant you return a Response;
pending promises die. `.catch()` alone doesn't help — that just attaches a
handler to a promise that never resolves. Symptom: booking succeeds, but no
emails are sent and Resend shows zero attempts.

### 2. Google Calendar API needs explicit per-project enablement
Easy to miss because OAuth and Calendar API are separate switches. Symptom:
`403 PERMISSION_DENIED` in logs with message "Google Calendar API has not
been used in project X before or it is disabled." Fix: visit
`https://console.developers.google.com/apis/api/calendar-json.googleapis.com/overview?project=<NUMBER>`
and click Enable.

### 3. Consumer Gmail can't auto-generate Meet links
`conferenceData.createRequest` works reliably only on Google Workspace.
Personal `@gmail.com` accounts get the event created but `hangoutLink` comes
back empty indefinitely. No amount of retry fixes this — Google itself doesn't
support it. For consumer accounts, attach a static Meet URL via
`conferenceData.entryPoints` (using `STATIC_MEET_URL` env var) directly.

### 4. Google's account picker auto-selects the wrong account
When OAuth-ing on a system with multiple Google accounts signed in, the picker
tends to default to the most recently used personal Gmail. Build a "Connected
as" diagnostic that reads the email from the OAuth `id_token` (requires
`openid email` scopes) and surfaces it in the dashboard. Don't trust user
assertions about which account they selected.

### 5. OAuth Testing mode = 7-day refresh token expiration
Testing-mode OAuth apps have refresh tokens that expire after 7 days. The
integration silently breaks. Publish the app (Audience → Publish app) — the
"unverified app" warning is harmless because only the host OAuths (visitors
never see it).

### 6. Workspace admin must trust external OAuth Client IDs
If the OAuth client lives in a Cloud project owned by a different account than
the host's Workspace (e.g. personal Gmail's project), the Workspace admin must
explicitly trust the Client ID at admin.google.com → Security → API Controls
→ Manage Third-Party App Access. Otherwise advanced operations (like Meet
creation) may silently no-op even though OAuth grants succeeded.

### 7. `requestId` for createRequest should be alphanumeric
The slot ID has colons and dots from the ISO timestamp. Some Google APIs
silently no-op on non-alphanumeric identifier fields. Strip with
`booking.slotId.replace(/[^a-zA-Z0-9]/g, '')` before embedding.

### 8. Time-zone math without dependencies
You can implement DST-correct TZ conversion using just `Intl.DateTimeFormat`.
Format a UTC Date in the target TZ to get the wall-clock parts; the gap
between rendered and original gives you the offset. No date-fns / luxon needed.

### 9. Slot duration vs cadence
Two separate config values. Meeting duration (`SLOT_MINUTES`) is what gets
attached to the Calendar event. Slot cadence (`SLOT_INTERVAL_MINUTES`) is how
often new slots are offered. Buffer = cadence - duration. Don't conflate.

### 10. Atomic slot claim is non-negotiable
Use `SET NX` (set-if-not-exists) on the slot key. Two visitors clicking submit
on the same slot at the same time → one wins, one gets a clean 409 "slot
taken" response and is told to pick another time. Without atomic claim, you
get double-bookings.

### 11. Read the host's calendar via freebusy — don't try to maintain a copy
`calendar.freebusy.query` on `primary` returns `{ busy: [{start, end}, ...] }`
for whatever time range you ask. Cheap, privacy-respecting (no event details
exposed), and includes events from ALL sources — not just events you created.
That means the host blocks time by creating a Calendar event the way they
normally would. No separate "block off" admin UI required. Cache the result
in KV for ~60s to keep API usage trivial even under load.

Key scope note: `calendar.events` is NOT sufficient for freebusy. Creating
events needs `calendar.events`; `freebusy.query` needs the separate
`calendar.freebusy` scope. Request both. Without the second one the query
returns 403, and if your code swallows that error the host's calendar blocks
silently stop removing slots. Don't add `calendar.readonly` unless you need
to read event details (which leaks data you don't actually need).

### 12. Fail closed when availability is unknown
The tempting version of `getBusyRanges` catches every error and returns `[]`.
That fails OPEN: one expired token or missing scope and the page offers every
slot as free, letting visitors book over real commitments, with nothing
visibly wrong. Instead:

- Not connected at all → `[]` is fine (availability rules only, by design)
- Connected but the call failed → serve the last cached window if you have
  one, otherwise throw and show no slots
- On 401/403, log loudly that the token likely lacks `calendar.freebusy` and
  the host needs to re-connect

A booking page with no slots for a few minutes is a minor annoyance. A
double-booked host is the failure this whole system exists to prevent.

---

## Customization — what's safe to change

The user brings their own style. Everything below is theirs to own:

- **Visual design.** Colors, fonts, animations, layout. Recommend scoping
  accent colors via a CSS variable override on the booking page wrapper so
  components inherit automatically.
- **Copy.** Page title, intro, success message, email bodies. Write it in
  the host's voice.
- **Schedule.** Edit `AVAILABILITY` in `config.ts`. Multi-window days (lunch
  blocks, focus blocks) are supported via multiple entries per day.
- **Slot duration / cadence.** Change `SLOT_MINUTES` and `SLOT_INTERVAL_MINUTES`.
- **Lead time / horizon.** Change `MIN_LEAD_MINUTES` and `BOOKING_HORIZON_DAYS`.
- **Event metadata.** Title, description in `EVENT` const.
- **Booking page URL.** Pick any path: `/book`, `/sessions`, `/open-windows`,
  `/calendar`. Update internal references.

## Customization — what NOT to change

These patterns are load-bearing — change them and things break:

- The KV atomic-claim pattern (`SET NX`)
- Awaiting email/Calendar calls in serverless routes
- Token-gated approve/reject links (don't replace tokens with predictable IDs)
- OAuth state CSRF token validation
- Separating meeting duration from slot cadence
- Sanitizing the `requestId` for Calendar createRequest
- Requesting `calendar.freebusy` alongside `calendar.events`
- Failing closed when the freebusy lookup errors

## v2 ideas (defer until v1 ships)

- Admin dashboard view of upcoming bookings (currently visible only via Vercel
  KV dashboard)
- Automated 24h reminder emails (needs a cron — Vercel Cron Jobs or external
  scheduler)
- Visitor-facing reschedule / cancel links in confirmation emails
- Multiple event types (architecture supports via type slug in keys/routes)
- Custom intake questions per event type
- Per-photo / per-event inquiry context (e.g. visitor clicks "Inquire" on a
  specific work and the booking form pre-fills with that context)
