# EVENTS.md

One section per milestone. Fill each one in order as you go.

The prediction in milestone 2 should be written before you run the replay. That is the
part a TA will ask about.

Keep it short and specific. Point at operations, call sites, and what a caller
gets back.

---

## Milestone 1: what runs in the request path, and what runs after it

### The operations table

One row per operation. Read the code, not the file
names. "As wired today" is what the code does now. "Your call" is what you
would do if this were your service. List up to eight operations, and remove
the rows you don't use.

| # | Operation | As wired today | Your call | Why (in one line) |
|---|---|---|---|---|
| 1 | Update the search index for a new booking | Direct synchronous call in `BookingService.book()`, before `store.save()` | Event-driven: update the index from a subscriber after the booking is saved | Search downtime should not block valid bookings. |
| 2 | Check booking conflicts | Direct call to `conflictingBooking()` in `book()` | Synchronous, before saving and returning success | Prevent overlapping bookings. |
| 3 | Send confirmation email | `EmailConfirmationHandler` subscribes to `booking.confirmed` | Event-driven; explicitly retry failed deliveries later | Email failure should not block valid bookings. |
| 4 | | | | |
| 5 | | | | |
| 6 | | | | |
| 7 | | | | |
| 8 | | | | |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**First operation and your call: search indexing.** I would update the search
index through a subscriber to `booking.confirmed`, after the booking is saved.
The caller needs an immediate confirmation and booking id from `book()`, but
does not need the booking to appear in search immediately. A few minutes of
search delay is acceptable when the caller already has clear confirmation.
Availability checks read the booking store rather than the search index.
With the current wiring, a search outage makes `book()` throw before saving
the booking. With event-driven indexing, an indexing failure would be recorded
as a dead letter while the caller still receives the saved booking. This bus
still runs subscribers synchronously; it isolates their exceptions, and failed
indexing would need an explicit retry because the bus does not retry on its own.

**Second operation and your call: conflict checking.** I would keep this
synchronous. The caller needs to know the room is free before receiving a
successful booking. `book()` checks the store through `conflictingBooking()`
and throws `BookingError` when a clash exists. If availability cannot be checked,
the request should fail rather than confirm a potentially overlapping booking.

### The incident

**The call behind `TICKET.md`.** In `src/service/bookingService.ts`,
`BookingService.book()`, line 94 calls
`this.searchIndex.indexBooking(booking, room, now)` before `this.store.save(booking)`.
When the search cluster is unavailable, this call throws
`SearchIndexUnavailableError`. The exception propagates to the caller of
`book()` instead of returning a booking. The booking is not saved, and the
subsequent audit record and confirmation event are not produced. This makes
search availability a prerequisite for booking even though booking validation
and conflict checks do not depend on the search index.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** Name what you expect
to change in the ledger, at the payment provider, and in the bus log. Write it
down, then run the demo.

### Before your fix

**The run.** Paste what `npm run demo:replay` printed without editing it. A TA
may ask you about it in milestone 3.

**Your prediction against the run.** Where they agree and where they do not.

### Your fix

**The diff.** Paste it.

**What your fix uses to decide what to do on a second delivery, and what it
does not protect.** One or two lines.

### After your fix

**The run.** Paste what `npm run demo:replay` printed on the fixed code.

**What stays the same across its three states.** Name the numbers you
compared.

---

## Milestone 3: a delivery that stops halfway

### The failure

**The run.** Run `npm run demo:failure` and paste what the delivery left
behind (the ledger, the provider, and the dead-letter list).

### Your recovery

**What, if anything, the failed delivery left undone.**

**The option you took and its evidence.** If you redelivered the
message, paste the state after the redelivery and the bus log line for that
message. If you left the message for a person to handle, write the runbook line
they would follow, and name which fields of the dead-lettered message they read to
follow it and what each one tells them.

### The other option

**What the option you did not take would have cost.** One line, in terms of who
does the work, when the organizer gets their money, and what has to be true for
it to be safe.
