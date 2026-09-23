# reservation-service: Smells and One Fix

Three smells, one fix, two proposals, one false positive. File and method names point at
the code; the suite and typecheck were green before and after the one change.

---

## Milestone 1: Three smells

### Smell 1

**The smell.** Duplication over reuse. `reportGenerator.ts` rebuilds two computations the
module already has. `ReportGenerator.priceOf` re-implements `ReservationManager.calculatePrice`
plus `applyDiscounts` step for step, with the five pricing constants copied under new names
(`PREMIUM_MULTIPLIER` became `PREMIUM_RATE_MULTIPLIER`, `LONG_BOOKING_MINUTES` became
`LONG_BOOKING_CUTOFF`, `EVENING_START_MINUTE` became `EVENING_CUTOFF`). `ReportGenerator.occupancy`
re-implements the window-clipping loop that `availability.freeMinutes` already has, which is why
`freeMinutes` has no caller anywhere. In classic vocabulary `priceOf` is also feature envy (it reads
only `room` and `booking` fields and none of its own), but the label that explains how it got here
is the agent one.

**Classic or agent-specific.** Agent-specific: duplication over reuse, caused by missing context.
The renamed constants are the tell. Same five numbers, different five names, is not copy-paste; it is
a second author rebuilding the rule from memory because `calculatePrice` was not in front of it.
The stronger evidence is that the booking being repriced already carries `priceCents`, computed by
the manager when it was created. An author with `types.ts` and `createBooking` in context would have
read the field instead of recomputing it.

**Where in the code.** `src/reportGenerator.ts`, `ReportGenerator.priceOf` (called from `revenue`)
and the five constants at the top of the file. The original lives in `src/reservationManager.ts`,
`ReservationManager.calculatePrice` and `applyDiscounts`. Secondary copy: `ReportGenerator.occupancy`
versus `src/availability.ts`, `freeMinutes`.

**The principle it violates.** One home per rule (the pricing policy has two), and Information
Expert: the booking knows what it was charged, so revenue should ask the booking, not recompute
from the room.

**What it makes expensive.** It is wrong today, not only in the future. `revenue()` reprices from
the *current* room, so raising a room's `hourlyRateCents` after bookings exist changes reported
revenue for money that was already charged. I checked: one 9:00 to 11:00 booking at 6000/hour has a
receipt of 12000 and a revenue line of 18000 once the rate is set to 9000. And because `revenue()`
looks each room up in the list it was handed, a `ReportGenerator` built with a partial room list
silently drops confirmed bookings (count 0, total 0). For the next change: a new pricing rule (a
weekend rate, say) has to be added in two files under two naming schemes; miss one and receipts
disagree with the finance report, and nothing in the suite compares the two across a rule change.

### Smell 2

**The smell.** Phantom complexity. `src/cache/` is a 79-line TTL cache (a config object, `withTtl`,
`disabled`, max-entry eviction, `invalidate`, `size`) whose only consumer is one `get` in
`ReservationManager.listBookingsForRoom`. Nothing anywhere calls `set`. Every `get` misses and the
method falls through to storage on every call, so the cache changes nothing observable. `set`,
`invalidate`, `size`, `withTtl`, and `disabled` have no callers; `get` has one, and it can never hit.

**Classic or agent-specific.** Agent-specific: phantom complexity, from free volume (a cache costs
the author nothing to write, so it wrote one nobody asked for, with eviction and config helpers on
top) and missing context (it never checked whether anything would populate the cache, or whether
the read path is hot). In classic terms it is speculative generality plus a hidden dependency:
`QueryCache.get` and `set` read `Date.now()` directly, a clock their signatures never mention.

**Where in the code.** `src/cache/queryCache.ts` (`QueryCache.get`, `QueryCache.set`),
`src/cache/cacheConfig.ts`, and the read site in `src/reservationManager.ts`,
`ReservationManager.listBookingsForRoom` (plus the `cache` field and the constructor line).

**The principle it violates.** Every line has to be tested, kept working, and fit into the next
reader's context (the cost side of free volume). And controllability: the one behavior the cache
has, expiry, depends on the system clock and cannot be driven from a test.

**What it makes expensive.** Today: every reader of `listBookingsForRoom`, and of
`formatDailySummary` which calls it, has to trace through the cache to learn it does nothing. That
is review cost paid on every read of the file. Tomorrow: it is a trap. The next author who
"finishes" the cache by adding the missing `set` gets a 30-second stale read, because
`createBooking` and `cancelBooking` never invalidate. `formatDailySummary` would then list a
cancelled booking as confirmed and count its price in the confirmed total, and the test for that
bug would need to control the clock, which this design does not allow.

### Smell 3

**The smell.** God class. `ReservationManager` owns the room registry, booking creation and
cancellation, the double-booking rule (`hasConflict`), the pricing policy (`calculatePrice`,
`applyDiscounts`, five constants), notification dispatch and a notification log, the cache, and all
presentation (`formatReceipt`, `formatDailySummary`, `formatClock`, `formatMoney`). Its own doc
comment lists five jobs. The presentation half is the sharpest edge: `formatReceipt` is both the
customer receipt and the email body (`dispatchNotification` passes it as the body), so one method
serves two audiences.

**Classic or agent-specific.** Classic. This is the shape from lecture (bookings, billing, email,
formatting in one class), and none of the three agent causes is needed to explain it: it is the
"first place you add to" pattern, each addition reasonable on its own. One agent-flavoured symptom
sits inside it: the notifier is built in the constructor through
`createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)` rather than received, unlike `storage`, which
is received. Because nothing outside can see the channel, the class grew a `notificationLog` and
`recentNotifications` just so a test could tell a message went out.

**Where in the code.** `src/reservationManager.ts`, the whole class. Specifically `formatReceipt`,
`formatDailySummary`, `formatClock`, `formatMoney` (presentation), `calculatePrice` and
`applyDiscounts` (pricing), `dispatchNotification` and `recentNotifications` (delivery
bookkeeping), and the constructor.

**The principle it violates.** Cohesion: one class, one reason to change. This one changes for
booking-rule, pricing, wording, delivery, and caching reasons. The constructor also breaks
controllability (the instantiation antipattern: it builds the notifier instead of receiving it).

**What it makes expensive.** A wording change to the receipt is an edit to the file that owns the
double-booking rule and the pricing policy. Because the receipt *is* the email body, changing what
customers see on screen silently changes what they get in the mail, and nothing in the suite reads
either string beyond `formatDailySummary` containing a room name and a total. A test that wants to
assert on the email cannot reach the `EmailChannel` (`sentMessages` has no caller anywhere for
exactly this reason) and has to settle for `recentNotifications`, which records only
`channel:recipient:subject`. Divergent change is the history this file will have: pricing,
formatting, email, and cache commits all land here.

---

## Milestone 2: One small fix

**Which smell you attacked.** Smell 1, the pricing copy in `ReportGenerator`. It is the one that
already produces wrong numbers, it sits on the path of the most likely change in a booking service
(a new pricing rule), and the fix is smaller than the other two: the correct value is already
stored on every booking, so the second implementation can be read instead of recomputed.

**What changed.** One file, `src/reportGenerator.ts`. `revenue()` now sums `booking.priceCents`,
the price the manager computed and the customer was shown, instead of `this.priceOf(room, booking)`.
`priceOf`, `durationOf`, the five copied constants, and the now-unused `Booking` import are
deleted. The report generator no longer contains a pricing policy at all; the policy has one home
again, `ReservationManager.calculatePrice`. The diff is two changed lines (the import and the one
sum) and 25 deleted lines, all in that file.

**What you deliberately did not touch.** Scope line: I removed the copy and nothing else in
`revenue()`.
1. The lookup-and-skip for a booking whose room is not in the generator's list stays, even though
   pricing was the only reason `revenue()` needed the room. Dropping it changes what `revenue()`
   counts, which is a policy question ("does a report over a subset of rooms count the rest?") that
   deserves its own one-line change and its own test.
2. The second copy named in Smell 1, `occupancy`'s clipping loop versus `freeMinutes`, stays. It
   produces the same numbers today, and folding it into `freeMinutes` means computing booked
   minutes as window minus free with a clamp in between, an inversion I would want a test for
   before trusting it.
3. `calculatePrice` stays on the manager instead of moving to its own module. That is Proposal B,
   not a duplication fix.

**How you know behavior is preserved.** `npm test` is green (39 of 39) and `npm run typecheck`
passes, with no test edited. What the suite actually pins: `reporting.test.ts` "totals revenue over
the window" asserts `totalCents` equals `morning.priceCents + evening.priceCents`, `averageCents`
is 11700, and `byRoom` is `{ r1: 23400 }`; "leaves cancelled bookings out of revenue" pins the
status filter. So the sums, the average, and the per-room map are covered. What it would not catch:
it never changes a room's rate and never hands the generator a partial room list, so it cannot tell
the old repricing behavior from the new stored-price behavior. That is the only place the two
differ, and it is the place where the old code was wrong (checked before the change: 12000 on the
receipt, 18000 in the report). It also does not cover the unknown-room skip, which is one reason I
left that alone.

---

## Milestone 3: Two proposals and one false positive

### Proposal A (not coded)

For Smell 2, the phantom cache.

**The problem.** `src/cache/` is a cache that is never written, read through one call in
`ReservationManager.listBookingsForRoom`, with a clock dependency nobody can control. It does
nothing today and turns into a stale-read bug the day someone adds the missing `set`, because no
write path invalidates.

**The decomposition.** Delete `src/cache/` and the `cache` field, constructor line, and `get`
branch in the manager; `listBookingsForRoom` returns `storage.findByRoom(roomId)`, which is what it
does today in effect. If a measurement ever shows the read path is hot, caching goes *below* the
manager, not inside it: a `CachingStorageProvider` that implements `StorageProvider`, wraps the real
one, and owns the freshness rule in the one place that sees every write. `save` and `update`
invalidate the affected room's key, `findByRoom` fills it, and the manager never learns a cache
exists. It takes a `now: () => number` in its constructor so expiry is testable. Where the rules
live: freshness with the writes, in the storage decorator; booking rules in the manager, unchanged.

**One cost.** The decorator has to know the query shape to invalidate correctly. A per-room key
works for `findByRoom`, but every new query on `StorageProvider` (a `findByOrganizer`, say) means
another key scheme in the decorator, and getting one wrong is exactly the stale read we were
avoiding. Deleting the current code also throws away 80 lines someone may have meant to finish. I
would take that trade today, because nothing in the module shows a hot read path and the current
code guards against nothing; I would build the decorator only when a profile says `findByRoom` is
the cost.

### Proposal B (not coded)

For Smell 3, the God class.

**The problem.** `ReservationManager` changes for five unrelated reasons (booking rules, pricing,
wording, delivery, caching) and builds its own notifier, so the email cannot be observed from
outside and the receipt and the email body are one method.

**The decomposition.** Four pieces. `ReservationManager` keeps only the use case: the room
registry, the id counter, `createBooking`, `cancelBooking`, the query methods, and the
double-booking rule. It sequences the steps and owns no formatting or pricing. A new `src/pricing.ts`
gets a pure `priceFor(room, start, end)` with the five constants, the single home for the pricing
policy (after the Milestone 2 fix there is exactly one caller to move). A new `src/receipts.ts`
gets pure functions over data, `formatReceipt(booking, roomName)`, `formatDailySummary(roomName,
bookings)`, `formatClock`, `formatMoney`, so a wording change touches a file with no rules in it.
The notifier becomes a constructor parameter,
`constructor(storage, notifier: NotificationChannel = new EmailChannel())`, which lets
`notificationLog` and `recentNotifications` go (a test passes a recording channel; the
`NotificationChannel` interface already allows it) and lets `notifierFactory.ts` go with them,
since it registers one channel and has no caller outside this constructor. Where the rules live:
booking rules in the manager, price rules in pricing, presentation in receipts, delivery in the
channel.

**One cost.** The receipt needs the room's display name, which lives in the manager's registry.
Either every caller of `formatReceipt` does the lookup and passes the name, or `receipts.ts` takes
a lookup function, an interface designed before its second consumer exists and likely to churn. The
split also turns the one file everyone knows into four with new names, and adds a constructor
parameter every fixture has to think about. I would do it if wording or pricing keep changing, both
plausible for a booking service. If the module is feature-complete, I would inject the notifier and
stop there, since that is the one piece that blocks a test today.

### The thing that looks smelly but is fine

**What it is.** `src/validation.ts`, `validateReservationRequest`. About fifty lines, eleven `if`
guards in a row, each returning early, on a request type TypeScript already checks. On a smell pass
it matches long method, and `typeof request.organizer !== 'string'` looks like phantom complexity
(a guard on a field the compiler already types as `string`).

**Why it is fine.** It is the module's one entry point for caller-supplied data, and its shape is
the shape of the job. Every guard is one independent rule that reads only `request` and `room`; the
order decides only which reason is reported first. The function is pure (no clock, no storage, no
globals) and returns a value instead of throwing, so it is fully controllable and observable, which
is why `validation.test.ts` can pin it with a table that has at least one row per guard plus the two
boundary acceptances. It is the single home every booking path calls, the Lab 3 lesson: a rule
that lives here is enforced, a rule that lives anywhere else is a path that skips it. Splitting it
into eleven named predicates would add eleven names and a combinator that owns no rule of its own.
The `typeof` guard is not phantom either: `ReservationRequest` objects arrive from outside the
compiler's reach (a JSON body, a form), and at a boundary a runtime check on a typed field is the
contract, not paranoia.

**What would flip your verdict.** Two changes. First, a second entry path that needs a subset of the
rules, for example a `rescheduleBooking` that needs the time and building rules but not the
organizer and attendee rules. At that point the list should split into request-shape rules and
building-policy rules so neither caller repeats the other. Second, rules that branch on room kind:
`room.premium === true && request.attendees < 2` is the first of these, and if a second and third
`if (room.<kind>)` appear, the function becomes type checks instead of polymorphism and the
room-kind rules should move behind a per-room policy. Either change would make one list the wrong
home; until then it is the right one.

---

## Also noticed, not written up

Inventory for a refactoring plan, in the order I would take them:

- Three copies of the interval-overlap predicate with three argument orders:
  `availability.isSlotFree`, `ReservationManager.hasConflict` (its exact negation), and
  `ReportGenerator.overlapsWindow`. Primitive obsession underneath: `(start, end)` travels as two
  bare numbers through six signatures, so a swapped call compiles. A cleaning-gap rule would have
  to change all three, or `findAvailableSlots` will advertise slots that `createBooking` rejects.
- `notifierFactory.ts` is speculative over-abstraction: a registry with one registered channel, a
  `ChannelName` union with one member, and no caller that can choose a channel.
- Convention drift on "not found": `getBooking` returns `undefined`, `cancelBooking` throws
  `BookingError`, `revenue` skips silently, `occupancy` substitutes the id for the name, and
  `createNotificationChannel` throws a plain `Error`.
- Dead exports: `freeMinutes`, `withTtl`, `disabled`, `QueryCache.invalidate`, `QueryCache.size`,
  `registeredChannels`, `EmailChannel.sentMessages`, and `StorageProvider.clear` have no callers.
- "Confirmed only" is filtered by hand in five places across the manager and the report generator.
