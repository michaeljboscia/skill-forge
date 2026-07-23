---
name: mx-ghl-calendars
description: GoHighLevel / HighLevel API v2 calendars and appointments for AI agents — get free slots, book appointments, timezone-correct start/end times, calendarId resolution, appointment status, avoiding double-booking, and slot availability. Load when building scheduling, booking, availability checks, or appointment automation in GHL/LeadConnector.
---

# GoHighLevel Calendars — Scheduling for AI Coding Agents

**Loads when you check availability, book appointments, or automate scheduling in GoHighLevel.**

## When to also load
- `mx-ghl-core` — auth, Version header, rate limits (always)
- `mx-ghl-contacts` — appointments attach to a contactId
- `mx-ghl-webhooks` — reacting to `AppointmentCreate`/`Update`/`Delete`

---

## Level 1: Patterns That Always Work (Beginner)

### 1. Always fetch free slots before booking
Booking a taken slot conflicts. Read availability first, then book a slot you got back.

```ts
const resp = await ghlFetch(
  `/calendars/${calendarId}/free-slots?startDate=${startMs}&endDate=${endMs}&timezone=${encodeURIComponent(tz)}`
).then(r => r.json());
// resp is a DATE-KEYED object, not a flat array:
//   { "2026-07-23": { slots: ["2026-07-23T14:00:00-04:00", ...] }, "2026-07-24": {...}, traceId: "..." }
const allSlots = Object.entries(resp)
  .filter(([k, v]) => k !== "traceId" && v && Array.isArray(v.slots))
  .flatMap(([, v]) => v.slots)
  .sort();                       // ISO strings sort chronologically
const next = allSlots[0];        // earliest offered slot
```

### 2. `startDate`/`endDate` for slots are epoch milliseconds
```ts
❌ BAD
`?startDate=2026-07-22`                 // string date → wrong/empty results

✅ GOOD
`?startDate=${Date.parse("2026-07-22T00:00:00Z")}&endDate=${Date.parse("2026-07-29T00:00:00Z")}`
```

### 3. Always pass an explicit timezone
Availability and the times you get back depend on timezone. Never rely on a server default.

```ts
// IANA tz, e.g. "America/New_York"
`/calendars/${calendarId}/free-slots?startDate=${a}&endDate=${b}&timezone=America/New_York`
```

### 4. Book with ISO 8601 times that carry an offset
```ts
✅ GOOD
await ghlFetch("/calendars/events/appointments", {
  method: "POST",
  body: JSON.stringify({
    calendarId, locationId, contactId,
    startTime: "2026-07-23T14:00:00-04:00",   // offset-aware ISO 8601
    endTime:   "2026-07-23T14:30:00-04:00",
    title: "Discovery call",
    appointmentStatus: "confirmed",
  }),
});
```

### 5. Resolve `calendarId` at runtime — per location
```ts
const { calendars } = await ghlFetch(`/calendars/?locationId=${locationId}`).then(r => r.json());
const calendarId = calendars.find(c => c.name === "Sales Demo").id;
```

---

## Level 2: Timezones Done Right (Intermediate)

### Timezone is the #1 source of scheduling bugs
- Slots are returned relative to the `timezone` you request.
- A naive `new Date("2026-07-23T14:00:00")` uses the *server's* local zone — almost never what you want.
- Store and reason in UTC; render/select in the contact's or calendar's zone.

| Symptom | Cause | Fix |
|---|---|---|
| Appointments an hour off | DST or missing offset | Use offset-aware ISO strings |
| Empty slots | wrong tz or non-epoch dates | pass IANA tz + epoch-ms range |
| Double-booked | booked without re-checking slots | fetch free-slots immediately before booking |

### Book the slot you were offered, not a computed one
Don't compute the next slot yourself. Take an actual returned free slot's start/end so it matches the calendar's slot interval and buffers. Derive `endTime` from the calendar's own `slotDuration`/`slotDurationUnit` rather than hardcoding 30 minutes.

### Team / round-robin calendars need the assigned user
On round-robin or multi-user calendars, a free slot belongs to a specific user. Pass `userId` when requesting free-slots (to scope availability) and set `assignedUserId` on the appointment to pin the resource that actually owned the slot — otherwise you can book a user who isn't free.

### Appointment status values
`new`, `confirmed`, `cancelled`, `showed`, `noshow`, `invalid` (set what the flow needs). Cancelling is a status/update, not necessarily a delete.

---

## Level 3: Concurrency & Reliability (Advanced)

### Two bookings can race for the same slot
Free-slots is a read; between reading and booking, someone else can take it. Handle the conflict on booking rather than assuming your earlier read still holds.

```ts
const res = await ghlFetch("/calendars/events/appointments", { method: "POST", body });
if (res.status === 409 || res.status === 422) {
  // slot taken since we checked → re-fetch free slots and offer alternatives
}
```

### Reschedule = update the same event
```ts
await ghlFetch(`/calendars/events/appointments/${appointmentId}`, {
  method: "PUT",
  body: JSON.stringify({ startTime, endTime }),
});
```
Don't cancel + create for a reschedule; it loses history and can double-notify.

---

## Performance: Make It Fast

- **Query the smallest date window you need.** Requesting a month of slots when you show a week wastes payload and time.
- **Cache calendarId per location**; it doesn't change between bookings.
- **Batch availability for a UI once**, then let the user pick, rather than re-querying on every interaction.

```ts
❌ BAD — refetch calendars for each booking
for (const b of bookings) { const id = await findCalendar(loc); await book(id, b); }

✅ GOOD
const id = await findCalendar(loc);
for (const b of bookings) await book(id, b);
```

## Observability: Know It's Working

- Track **booking success vs conflict (409/422) rate** — rising conflicts mean your free-slot read and booking are too far apart.
- Alert on **empty-slot responses** for calendars that should have availability (often a tz/date-format regression, not real unavailability).
- Log booked `startTime` in both UTC and display tz to debug off-by-an-hour reports fast.
- Watch `AppointmentCreate`/`Update` webhooks to reconcile bookings made outside your integration.

---

## Enforcement: Anti-Rationalization Rules

### Rule 1: Check free-slots before every booking
**You will be tempted to:** compute a start time and POST an appointment directly because it's fewer calls.
**Why that fails:** you'll book taken or invalid slots, ignore buffers/slot intervals, and generate conflicts.
**The right way:** fetch `/free-slots`, book a returned slot, and handle a conflict if the slot was taken meanwhile.

### Rule 2: Timezone is always explicit
**You will be tempted to:** omit `timezone` and use local `Date` math.
**Why that fails:** availability and stored times land in the wrong zone; appointments show up an hour off, especially across DST.
**The right way:** pass an IANA `timezone` to free-slots and use offset-aware ISO 8601 for start/end.

### Rule 3: Slot ranges are epoch milliseconds
**You will be tempted to:** pass `startDate=2026-07-22` as a date string.
**Why that fails:** the endpoint expects epoch ms; string dates return wrong or empty results and you'll think the calendar is empty.
**The right way:** `Date.parse(...)` / `.getTime()` to epoch ms for `startDate`/`endDate`.

### Rule 4: Don't hardcode calendarId
**You will be tempted to:** store one calendarId in config.
**Why that fails:** calendars are per-location; the ID is wrong in every other sub-account.
**The right way:** resolve by name from `/calendars/?locationId=` and cache per location.

### Rule 5: Reschedule via update, not cancel+create
**You will be tempted to:** cancel the old appointment and create a new one to "move" it.
**Why that fails:** it fires cancel + create notifications, loses the event's history, and can leave orphans on failure.
**The right way:** `PUT` the appointment's start/end to reschedule in place.
