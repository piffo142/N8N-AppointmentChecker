# n8n Workflow: Appointment Availability Check (BeautyDesk) — Final

> One calendar per salon. No resource fan-out needed.

## Trigger
**Node:** Webhook (POST)
- Path: `/appointment-enquiry`
- Auth: Header Auth (shared secret)
- Payload:
```json
{
  "salonId": "string",
  "serviceId": "string",
  "requestedDate": "2026-07-05",
  "requestedTime": "14:00",
  "durationMinutes": 45,
  "customerName": "string",
  "customerEmail": "string",
  "customerPhone": "string"
}
```

## Step 1 — Validate Input
**Node:** Code (JS)
- Reject past dates, malformed ISO date/time, missing `salonId`
- On failure → **Respond to Webhook** (400), stop

## Step 2 — Resolve Salon → Calendar
**Node:** Postgres/MySQL (or Airtable)
- Lookup table: `salonId` → `googleCalendarId`, `timezone`, `bufferMinutes`
- One row per salon, single calendar field — no per-staff/per-room mapping
- Output: `calendarId`, `timezone`, `bufferMinutes`

## Step 3 — Compute Requested Window
**Node:** Code (JS), using `luxon`
- Convert `requestedDate` + `requestedTime` to UTC via salon `timezone`
- `slotStart`, `slotEnd = slotStart + durationMinutes`
- Also compute `dayStart`/`dayEnd` (full business day, e.g. 09:00–18:00 local) — pull the whole day's busy blocks in one call so Step 5 can suggest alternatives without a second API round-trip

## Step 4 — Query Google Calendar
**Node:** HTTP Request → `freebusy.query`
```json
{
  "timeMin": "{{ $json.dayStart }}",
  "timeMax": "{{ $json.dayEnd }}",
  "items": [{ "id": "{{ $json.calendarId }}" }]
}
```
- Credential: OAuth2 per salon (see Auth Model decision below)

## Step 5 — Evaluate Availability
**Node:** Code (JS)
- Filter `busy[]` to blocks overlapping `[slotStart, slotEnd + bufferMinutes]`
- `available = overlaps.length === 0`
- If unavailable: scan `dayStart`→`dayEnd` in fixed increments (e.g. 15 min) against the full `busy[]` array, return first 3 open slots ≥ `durationMinutes` + buffer

## Step 6 — Branch
**Node:** IF (`available === true`)
- **True:**
  - Write a pending-hold event to the calendar (5–10 min TTL) to prevent a race with a concurrent enquiry for the same slot. Set `extendedProperties.private.type = "pending_hold"` and `extendedProperties.private.expiresAt = <ISO timestamp>` at creation — required for the cleanup workflow below to identify holds reliably (don't rely on title parsing)
  - Respond 200: `{ available: true, slot: { start, end }, holdEventId }`
- **False:**
  - Respond 200: `{ available: false, suggestedSlots: [...] }`

## Step 7 — Respond to Webhook
**Node:** Respond to Webhook
- JSON mirror of the branch output

---

## Workflow Diagram

```
Webhook
  │
  ▼
Validate Input ──(invalid)──► Respond 400
  │ (valid)
  ▼
Resolve Salon → Calendar (DB lookup)
  │
  ▼
Compute Requested Window (Code, luxon)
  │
  ▼
Google Calendar freebusy.query (full day)
  │
  ▼
Evaluate Availability (Code)
  │
  ▼
IF available?
  ├─ true  ──► Create pending hold event (tagged) ──► Respond 200 {available:true, slot, holdEventId}
  └─ false ──► Compute suggested slots ──► Respond 200 {available:false, suggestedSlots}
```

## Resolved Design Decisions
- **Race condition:** Pending-hold event, 5–10 min TTL, created on the salon's calendar when `available: true` is returned. Cleanup handled by a separate scheduled workflow (below) — orphaned holds would otherwise silently eat availability.
- **Auth model:** Per-salon OAuth2, default-safe given mixed/unknown Workspace status across salons. Means each salon's onboarding flow needs an OAuth consent step against their own Google account — flag for the BeautyDesk onboarding UX, not just the n8n credential store. Token refresh failures need a salon-facing alert (see Token Failure Alert workflow below), or appointment checks silently start failing.
- **Voice channel reuse:** Inline for MVP. Steps 3–5 stay embedded in this workflow. When Vapi/Twilio voice booking lands, extract into a sub-workflow at that point — don't over-engineer the abstraction now on spec.

## Open Items Still Requiring Decisions
1. **Confirm-step interaction:** when a customer confirms, that flow must either delete the hold event and create a real booking, or convert the hold in-place (clear `pending_hold` extended property, set real event details). Converting in-place avoids a delete+recreate race against the cleanup workflow running mid-confirm.
2. **Buffer policy:** currently flat per-salon (Step 2). Per-service buffer override (e.g. colour treatments need cleanup time, a blow-dry doesn't) is a v2 concern.
3. **Reconnect UX:** detecting and alerting on auth failure (below) still requires a human or admin-UI flow to actually re-authorize. Decide whether that's v1 scope or a manual ops task while BeautyDesk is small.

---

# n8n Workflow: Expired Hold Cleanup (BeautyDesk)

## Trigger
**Node:** Schedule Trigger
- Interval: every 5 minutes
- No payload — runs across all salons in one pass

## Step 1 — Fetch All Salons
**Node:** Postgres/MySQL → "Select"
- Query: all rows from salon config table (`salonId`, `googleCalendarId`, `timezone`)

## Step 2 — Loop Per Salon
**Node:** Split In Batches (batch size 1, or higher if volume allows parallel-safe calls)

## Step 3 — List Pending Hold Events
**Node:** Google Calendar → "Get Many" (Events)
- Calendar: `{{ $json.googleCalendarId }}`
- Filter: `extendedProperties.private.type = "pending_hold"` (set at creation time in the availability-check workflow's Step 6)
- Time range: `timeMin = now - 1h`, `timeMax = now + 1h` (keeps the query cheap)

## Step 4 — Filter Expired
**Node:** Code (JS)
- Read `extendedProperties.private.expiresAt` per event
- Keep only events where `expiresAt < now`

## Step 5 — Delete Expired Holds
**Node:** Google Calendar → "Delete Event"
- Loop over filtered list, delete by `eventId`
- Wrap in try/catch — a 404 (already deleted/confirmed) should skip and continue, not fail the batch

## Step 6 — Loop Back
**Node:** Split In Batches → loop until all salons processed

## Step 7 — Log Summary (recommended)
**Node:** Code → write to a `cleanup_log` table or notification channel
- Count deleted per salon, flag any salon with unusually high hold counts (could indicate a stuck confirm flow upstream)

---

## Diagram
```
Schedule (every 5 min)
  │
  ▼
Fetch All Salons (DB)
  │
  ▼
Split In Batches ◄────────────┐
  │                           │
  ▼                           │
List Pending Holds (Calendar) │
  │                           │
  ▼                           │
Filter Expired (Code)         │
  │                           │
  ▼                           │
Delete Expired (Calendar) ────┘
  │
  ▼
Log Summary
```

## Design Notes
- **Critical dependency:** the availability-check workflow must write `extendedProperties.private.type` and `expiresAt` when creating the hold. Without this tag, cleanup can't distinguish a hold from a real confirmed booking — deleting the wrong event type is a production incident waiting to happen.
- **Confirm-step interaction:** see Open Items #1 above — converting holds in-place on confirm avoids a race against this job.

---

# n8n Workflow: OAuth2 Token Failure Alert (BeautyDesk)

## Trigger
**Node:** Error Workflow (n8n's built-in workflow-level error trigger)
- Attach as the **Error Workflow** on both the availability-check and cleanup workflows (set in each workflow's Settings → Error Workflow)
- Fires on any node failure in those workflows — Step 1 below filters for auth-specific errors

## Step 1 — Filter for Auth Failures
**Node:** IF / Code
- Inspect the error payload for Google API codes: `401`, `403` with `invalid_grant`, `unauthorized_client`, or token-expired messages
- Non-auth errors → route elsewhere (general error log) or no-op

## Step 2 — Identify Affected Salon
**Node:** Code (JS)
- Extract `salonId` / `calendarId` from the failed node's input data, available in the error trigger's execution context

## Step 3 — Check Alert Throttle
**Node:** Postgres/MySQL → Select, then IF
- Query `alert_log`: has an auth-failure alert already fired for this `salonId` in the last 24h?
- If yes → stop (avoid spamming the salon owner every 5 minutes when the cleanup workflow keeps hitting the same expired token)
- If no → proceed

## Step 4 — Log the Alert
**Node:** Postgres/MySQL → Insert
- Write `salonId`, `timestamp`, `errorType` to `alert_log`

## Step 5 — Notify
**Node:** Send Email (or Slack/Twilio SMS depending on BeautyDesk's actual ops channel)
- To: salon's registered contact email (from salon config table)
- Also: internal ops channel — visibility across the customer base, not just left to the salon to notice
- Message: plain language, "Your calendar connection needs to be reconnected" + link into the BeautyDesk admin UI's reconnect flow (implies a one-click reconnect entry point is needed in the admin UI — flag for product scope if it doesn't exist yet)

## Step 6 — Mark Salon as Degraded (recommended)
**Node:** Postgres/MySQL → Update
- Set `salon.calendarStatus = 'disconnected'` on the salon config row
- Availability-check workflow can check this flag upfront (cheap, before attempting the Calendar call) and fail fast with a clear customer-facing message instead of a generic error per enquiry

---

## Diagram
```
Error Workflow Trigger (attached to availability-check + cleanup workflows)
  │
  ▼
Filter: is this an auth failure? ──(no)──► (ignore / general log)
  │ (yes)
  ▼
Identify Affected Salon
  │
  ▼
Check Alert Throttle (24h) ──(already alerted)──► stop
  │ (not yet alerted)
  ▼
Log Alert
  │
  ▼
Notify (email salon + internal ops)
  │
  ▼
Mark salon.calendarStatus = 'disconnected'
```

## Design Notes
- **Reconnect flow is the real gap** — this workflow only detects and alerts. Someone (salon owner, via BeautyDesk admin UI) still has to re-auth. Decide whether that's v1 requirement or a manual ops task while BeautyDesk is small.
- **`calendarStatus` flag** ties this workflow to the availability-check workflow — add a "check salon.calendarStatus before calling Calendar API" step there when this piece is implemented, not as a separate later task.
