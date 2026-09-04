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
  Parent Zap  ── catch hook ──▶ create Zapier Table record
       │
       ▼
  Sub-Zap "Process Panetta Booking Logic"   ← documented below
       │
       ├─ Path A: free    → calendar event, status "Pending Approval", success email
       └─ Path B: busy    → status "Time Conflict", conflict email
       │
       └─ error: calendar query failed → status "Time Conflict", invalid-times email

  Approval Zap  ── table record updated ──▶ approved email
```

Two Zaps plus a Sub-Zap. The approval Zap is separate and fires when an
administrator changes a record's status.

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

The approval Zap's trigger exposes fields as `old.data.fN` rather than by name:
`f1` email, `f2` name, `f17` event title, `f18` start, `f22` end, `f24`
responsible party, `f25` total guests. `f18` and `f22` are read as `["label"]`,
not `["value"]`. There is no way to tell from the repository what any other `fN`
holds.

## Sub-Zap: Process Panetta Booking Logic

Validates a reservation and either books it or turns it away. Triggered by the
parent Zap after a booking record is created.

### Input

Nine fields from the parent: Responsible Party, Total Guests, Date, Start Time,
End Time, Event Title, Record ID, Recipient Email, Recipient Name.

The form sends **30**. The rest sit in the table record for an administrator to
read at approval time — the description, the guest breakdown, food, vendor and
alcohol answers, the three agreement confirmations, and the free-text
"additional dates" field.

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
