# Arts Division room reservations

Booking forms for UCSC Arts Division event spaces. A request checks the room's
live Google Calendar availability, then posts to a Zapier webhook that creates
the calendar event and sends the email. Every request is approved by an
administrator before it is confirmed.

Live at <https://ghagenah.github.io/reservations/>

## Layout

```
index.html          redirect to panetta/
styles.css          all form styling, shared by every space page
reservation.js      all form behaviour, shared by every space page
confirmation.html   landing page after a successful request, shared
guide.html          event guide (Panetta-specific)
assets/             photos, and the email banner

panetta/index.html  Panetta Conference Room — one room
darc/index.html     Digital Arts Research Center — two rooms
```

`/reservations/` in the public URL comes from the repository name, not from a
folder here.

## How a space works

A space page is markup plus a `window.SPACE_CONFIG` block declaring what is
specific to it. Everything else — styling, validation, availability, the
submit flow — is in `styles.css` and `reservation.js` and is shared.

```js
window.SPACE_CONFIG = {
  spaceName: 'Panetta Conference Room',
  webhookUrl: 'https://hooks.zapier.com/hooks/catch/...',
  rooms: [
    { label: 'Panetta Conference Room', calendarId: '...@group.calendar.google.com' }
  ],
  draftKey: 'panetta-reservation-draft',
  confirmationPage: '../confirmation.html',
  maxGuests: 73,
  maxGuestsNote: '73 is the highest listed occupancy, for standing room.'
};
```

One room means no room picker and the form reads as a single-room booking.
More than one builds a radio group from `rooms`, requires a choice, and keys
availability to the chosen room's calendar.

Each room needs its own Google Calendar, made **public** — the API key can
only read public calendars.

## Adding a space

1. Copy an existing space folder, e.g. `cp -r panetta newspace`.
2. Edit its `SPACE_CONFIG`: name, webhook, rooms, a **unique** `draftKey`,
   guest cap. Leave `confirmationPage` and the `../` paths alone.
3. Replace the space-specific copy: title, subtitle, hero image, occupancy
   table, checklist wording.
4. Give it a Zap of its own. A multi-room space should branch on the `room`
   field to choose which calendar to write to.

Two spaces sharing a `draftKey` would share saved drafts. Nothing checks for
this.

## Adding a field

Five places, all following the pattern of any existing field:

1. The markup, in each space page that needs it
2. An element reference near the top of `reservation.js`
3. A rule in `fieldErrors()` if it is required
4. An entry in `payload` in `handleSubmit()`
5. The mapping in Zapier, and the email template if it should appear there

Add it to `NEVER_RESTORE` if it is a consent checkbox — those are deliberately
not restored from a saved draft.

## Things worth knowing

- **Times are Pacific**, always, whatever clock the visitor is on. The helpers
  in `reservation.js` convert via `Intl` so DST is handled; do not add or
  subtract hours by hand.
- **The submission is form-encoded, not JSON.** Zapier's catch hook answers a
  CORS preflight without `Access-Control-Allow-Headers`, so a JSON content type
  gets the POST blocked by the browser before it is sent.
- **The Google API key is public** and restricted by HTTP referrer to
  `ghagenah.github.io/*`. It is read-only against public calendars. Requests
  from `localhost` are rejected, so availability will not load when testing
  locally — everything else does.
- **Availability is prefetched** for 45 days and cached for 5 minutes, then
  re-checked live at submit time in case the hours were taken while the form
  was being filled in.
- **Drafts survive submission** on purpose, so someone turned away by a clash
  only has to pick a new time rather than retype everything.
- `styles.css` is not used by `confirmation.html` or `guide.html`. Those define
  `.card`, `body`, `h1` and `.hero` differently and keep their styling inline.
  The colour palette is therefore repeated in three files.

## Testing

Add `?demo=1` to a space URL for a button that fills the form with plausible
test data and picks a real open time slot. Titles are prefixed `TEST`.
Submitting still creates a real calendar event and sends real email.

## Email templates

`email-templates/` holds the four Zapier outbound emails: request received,
approved, time-slot conflict, invalid times. They are pasted into Zapier by
hand, so a change here is not live until it is pasted. The merge expressions
reference Zap step IDs and will not survive reformatting.

## Outstanding

- `darc/index.html` is not live: its webhook and both calendar IDs are
  `REPLACE_` placeholders, occupancy is `TBD`, and it has no photo. Grep
  `REPLACE_` and `TODO`.
- `guide.html` is Panetta-specific. A second space needing a guide means
  deciding whether it becomes per-space.
- The Zapier side is undocumented, and is the part of this system that cannot
  be read from the repository.
