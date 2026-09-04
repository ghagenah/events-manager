# The Zapier side

The half of this system that is not in this repository. The form posts to a
webhook and everything after that — the calendar event, the table record, the
four emails — happens in Zapier, where it can only be read by clicking through
the editor. This file is the map.

See [README.md](README.md) for the front end.

## Shape

```
form (this repo)
  └─ POST, form-encoded, 30 fields
       │
       ▼
  [PARENT] new booking      ── catch hook ──▶ create table record ──┐
                                                                   │
  [PARENT] rescheduled      ── record updated ──▶ filter ──▶ delete │
           booking                                    old event ───┤
                                                                   │
  [PARENT] approval         ── record updated ──▶ approved email    │
                                                                   ▼
                            [CHILD] Process Panetta Booking Logic
                                       │
                       ├─ Path A: free  → event, "Pending Approval", success email
                       ├─ Path B: busy  → "Time Conflict", conflict email
                       └─ error         → "Time Conflict", invalid-times email
```

Three parent Zaps and one child. Two of the parents trigger on the *same*
event — any update to a record in the Panetta Reservations table — and are
separated only by their filters.

## Step IDs

The email templates address Zap steps by numeric ID. These are not editable
text — renaming or reformatting a merge expression breaks the send.

| ID | What it is | Read by |
|---|---|---|
| `373784826` | Sub-Zap trigger (Start a Sub-Zap) | success, conflict, invalid |
| `377256141` | Date formatter, Step 2 | success, conflict, invalid |
| `377074420` | Create Google Calendar Event, Path A | success only |
| `377233437` | Approval Zap trigger — table record updated | approved only |
| `374556109` | Google Calendar lookup in the approval Zap | approved only |

`377074420` is read only by the success email because it is the only path where
a calendar event exists.

Table fields are addressed as `fN` rather than by name. Known so far:

| Field | Holds | Seen in |
|---|---|---|
| `f1` | recipient email | approved email |
| `f2` | recipient name | approved email |
| `f7` | Google Calendar event ID | reschedule Zap |
| `f17` | event title | approved email |
| `f18` | start time — read as `["label"]`, not `["value"]` | approved email |
| `f22` | end time — read as `["label"]` | approved email |
| `f24` | responsible party | approved email |
| `f25` | total guests | approved email |

Nothing in the repository says what any other `fN` holds. Add to this table
when you find out.

## Sub-Zap: Process Panetta Booking Logic

Validates a reservation and either books it or turns it away. Called by two
parents: after a new booking record is created, and after an existing one is
rescheduled.

### Input

Two different parents call this child, and **they do not pass the same fields**:

| From new booking | From reschedule |
|---|---|
| Date, Start Time, End Time, Event Title, Record ID, Recipient Email, Recipient Name | same seven |
| Responsible Party, Total Guests | — |
| — | Start DateTime, End DateTime |

That difference is a live bug — see *Known problems*.

The form sends **30** fields in all. The rest sit in the table record for an
administrator to read at approval time: the description, the guest breakdown,
food, vendor and alcohol answers, the three agreement confirmations, and the
free-text "additional dates" field.

### Steps

1. **Queue delay, 0.25 min.** Serialises simultaneous submissions so two
   requests cannot both find the same hour free. See *Three layers of conflict
   detection* below for why this is not the only guard.
2. **Format date** to `dddd, MMMM D, YYYY`. Every email reads this rather than
   formatting a date itself.
3. **Check availability** — query the Panetta calendar for busy periods between
   the requested start and end. Wrapped in an error handler.
4. **Branch.**

**Path A — free.** Create the calendar event (title, date, time, location `100
Panetta Ave, Santa Cruz, CA 95060`, attendee) → update the record to
`Pending Approval` and store the calendar event ID → send the *request
received* email → return `Approved`.

**Path B — busy.** Update the record to `Time Conflict` → send the *time slot
unavailable* email → return `Conflict`.

**Error — the calendar query itself failed.** Update the record to
`Time Conflict` → send the *invalid times* email.

### Return

```json
{ "status": "Approved" | "Conflict", "message": "…" }
```

Note that `Approved` here means "added to the calendar", not "an administrator
approved it". The record's status at this point is `Pending Approval`.

## Parent: Rescheduled Booking Request

Triggered when a record in the table is updated. Deletes the old calendar
event, if there was one, and re-runs the booking logic for the new time.

1. **Trigger** — Zapier Tables, record updated.
2. **Queue delay, 0.25 min** — collapses a burst of quick edits to the same
   record into one run.
3. **Find Record** by Record ID, for the full current row.
4. **& 5. Strip the time off** the new and old dates, so the comparison below
   is date-to-date rather than timestamp-to-timestamp.
6. **Filter** — continue only if the time or date actually changed. Editing a
   title or an email address should not touch the calendar. **See *Known
   problems*: as described, this filter looks inverted.**
7. **Branch on whether a calendar event exists** — old `f7` non-empty means
   delete the old event first (silently, no attendee notification), then call
   the child. Empty `f7` means no event was ever created, most likely because
   the original request hit a conflict, so it calls the child directly.

Both branches then call the child Sub-Zap with the new details.

Rescheduling therefore sends the requester the *request received* email again
and puts the record back to `Pending Approval` — an already-approved booking
that gets moved silently needs approving a second time. That is defensible, but
it is not obvious from the outside.

## Record status

```
Pending Approval  ──(administrator)──▶  Approved  ──▶  approval Zap sends approved email
Time Conflict     (dead end — the requester is asked to submit again)
```

## Three layers of conflict detection

The form checks availability too, which is why a conflict email exists at all
and why it should be rare:

1. **Prefetch**, 45 days, cached 5 minutes — what the slot grid is drawn from.
2. **Re-check at submit**, live, about a second before the POST. Catches
   anything booked while the form was being filled in. Fails open: if the
   check cannot run the submission proceeds.
3. **This Sub-Zap**, after the queue delay. The last word, and the only one
   that can see another request already in flight.

Layer 3 is what the conflict email reports. If it fires often, something is
wrong with layer 1 or 2, not with the room being busy.

## Known problems

**The error path sends the wrong email.** A failed calendar query sends the
*invalid times* template, which tells the requester "the start and end times did
not make sense together — most often the end time falls before the start time."
That is one possible cause. A Google outage, an expired connection, or a
permissions change produce the same email, and the requester is told they made a
mistake they did not make. Either the copy should describe a system fault, or
the error path should distinguish a genuinely malformed time range from an API
failure.

**The error path also sets the wrong status.** A calendar failure records
`Time Conflict`, which is indistinguishable in the table from a real
double-booking. Nobody can tell how often the calendar query is failing.

**A rescheduled booking sends an email with two blank rows.** The success and
conflict templates both read `Responsible Party` and `Total Guests` from the
child's trigger, but the reschedule parent does not pass either. Every email
sent after a reschedule renders those two rows empty. Fix by passing both from
the reschedule parent — the values are already on the record it just looked up,
as `f24` and `f25`.

**The reschedule filter looks inverted.** Described as three conditions that
must *all* be false to continue — new date equals old, new start equals old, new
end equals old — which is an AND, so the Zap proceeds only when the date **and**
the start **and** the end have all changed. The intent is surely *any* of them.
As written, the ordinary cases stop:

| Change | date same? | start same? | end same? | proceeds? |
|---|---|---|---|---|
| Tuesday 9–11 → Wednesday 9–11 | no | **yes** | **yes** | no |
| 9–11 → 10–11, same day | **yes** | no | **yes** | no |
| 9–11 → 9–12, same day | **yes** | **yes** | no | no |
| Tuesday 9–11 → Wednesday 10–12 | no | no | no | yes |

Moving a meeting to a different day at the same time is the single most likely
reschedule, and it is the first row. The symptom is silent: the record updates,
no calendar change happens, and the old event stays where it was. Worth checking
in the editor before anything else here — this is read from the description
rather than from the Zap, and Zapier filters do support OR groups, so it may
already be built that way.

**Three timezone spellings are in play.** The form works in
`America/Los_Angeles`, the reschedule Zap in `PST8PDT`, and the calendar's own
events are labelled `America/New_York`. The first two agree in practice —
`PST8PDT` follows the same US DST rules — so nothing is broken, but one spelling
would be easier to trust.

**The calendar's timezone is America/New_York.** Every event on it carries that
label while its times carry correct Pacific offsets, so the instants are right
and nothing is visibly broken. It is why an embed without an explicit `ctz`
shows Eastern times. Worth setting to `America/Los_Angeles` in Google Calendar
settings, and worth doing before anything recurring is ever added, because
recurrence is expanded in the event's own timezone.

## Worth changing

**Use the ISO timestamps.** The form already sends
`startTimeISO` / `endTimeISO` as `2026-08-31T11:00:00-07:00` — a full instant
with the Pacific offset, DST included. The Zap currently recombines a date with
plain-text times like `11:00 AM`, which is more parsing and more that can go
wrong. `timeZone` is sent too.

**DARC will need its own Zap.** The DARC form has two rooms and sends `room`
(`Room 1` / `Room 2`) and `location` for exactly this. Branch on `room` to pick
which calendar to check and write to. Note the Sub-Zap above is
Panetta-specific throughout — the calendar, the hardcoded street address, the
email copy.

## Editing the emails

The four templates live in `email-templates/` and are **pasted into Zapier by
hand**. A change in this repository is not live until someone pastes it. There
is no automation and no warning if they drift.

| File | Sent when |
|---|---|
| `success.html` | Path A — request received, time held |
| `conflict.html` | Path B — hours already booked |
| `invalid.html` | Error path — see *Known problems* |
| `approved.html` | Approval Zap — administrator approved it |
