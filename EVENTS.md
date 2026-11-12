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

**What you expect `redeliver` on a refund message to do.** I expect redelivery not to add another ledger entry or issue another refund, with the same message remaining in the bus log and its delivery count reaching 3 after two redeliveries.

### Before your fix

**The run.**

```text

> campus-reservations@1.0.0 demo:replay
> tsx src/demos/replayDemo.ts

booked bk-1001 for 8500 cents
cancelled bk-1001, refund 8500 cents, refund message msg-3

=== after the cancellation (one delivery of the refund message) ===
ledger entries: 1
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 1
gateway settlements: 1
  stl-1 bk-1001 $85.00
money returned in total: $85.00
dead letters: 0

redelivering msg-3

=== after one redelivery ===
ledger entries: 2
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 2
gateway settlements: 2
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
money returned in total: $170.00
dead letters: 0

redelivering msg-3 again

=== after two redeliveries ===
ledger entries: 3
  led-1 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-2 bk-1001 $85.00 (cancelled more than two hours ahead)
  led-3 bk-1001 $85.00 (cancelled more than two hours ahead)
gateway calls: 3
gateway settlements: 3
  stl-1 bk-1001 $85.00
  stl-2 bk-1001 $85.00
  stl-3 bk-1001 $85.00
money returned in total: $255.00
dead letters: 0

=== bus log ===
  msg-1 booking.confirmed deliveries=1
  msg-2 booking.cancelled deliveries=1
  msg-3 refund.requested deliveries=3
```

**Your prediction against the run.** 

The delivery count matched my prediction, but each delivery added an entry and refunded another $85, bringing the total to $255 instead of staying at $85.

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
