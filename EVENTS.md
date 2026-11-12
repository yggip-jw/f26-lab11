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
| 4 | Save the booking | Direct call to `store.save()` in `book()` | Synchronous, before returning success | A booking must be saved before it is confirmed. |

### Two calls in detail

Pick two rows and give each a paragraph rather than a line. One of them has to
be the operation `TICKET.md` implicates.

**Search indexing.** I would use a `booking.confirmed` subscriber so callers immediately receive the saved booking and id even when search is unavailable, accepting delayed search visibility and explicitly retrying indexing later.

**Conflict checking.** I would keep `conflictingBooking()` synchronous so callers know the room is free before success, with conflicts causing `book()` to throw `BookingError`.

### The incident

**The call behind `TICKET.md`.** In `src/service/bookingService.ts:94`, `BookingService.book()` calls `this.searchIndex.indexBooking(booking, room, now)` before saving, so a search outage propagates `SearchIndexUnavailableError` to the caller instead of saving and returning a booking.

---

## Milestone 2: what a second delivery does

### Your prediction (before you run anything)

**What you expect `redeliver` on a refund message to do.** I expect redelivery not to add another ledger entry or issue another refund.

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
