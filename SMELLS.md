# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplicated pricing rule (shotgun-surgery risk). The price formula exists twice:
once when a booking is created and again, independently, when revenue is reported. The report
ignores the `priceCents` already stored on every `Booking` and re-derives it.

**Classic or agent-specific.** Agent-specific: *duplication over reuse*. Classic duplication
accretes over years as people copy a block and tweak it. Here the second copy is complete and
consistent from the first commit, and the first copy's helper was sitting in the sibling file.
Nobody reached for it, because the second file was written in one pass without looking at
`ReservationManager`. A secondary agent-specific signal is *convention drift* inside the copy:
the five constants are re-declared under different names (`PREMIUM_MULTIPLIER` vs
`PREMIUM_RATE_MULTIPLIER`, `LONG_BOOKING_MINUTES` vs `LONG_BOOKING_CUTOFF`,
`EVENING_START_MINUTE` vs `EVENING_CUTOFF`), so even a search for one name will not find the
other copy.

**Cause (from the lecture).** Missing context. The author of `reportGenerator.ts` did not have
`ReservationManager.calculatePrice` in front of them, so the pricing rule was rebuilt from
scratch and renamed. Nothing about the request forced the duplication.

**Where in the code.** [src/reportGenerator.ts](src/reportGenerator.ts) `ReportGenerator.priceOf`
(lines 104-117) and constants (lines 4-8), duplicating
[src/reservationManager.ts](src/reservationManager.ts) `ReservationManager.calculatePrice` and
`applyDiscounts` (lines 140-158) and constants (lines 11-15).

**The principle it violates.** DRY, meaning one authoritative home for each business rule. A
second violation follows from the first: the report has no single source of truth for what a
booking cost, because the receipt shows `booking.priceCents` and the report shows
`priceOf(...)`.

**What it makes expensive.** Changing a price rule, for example raising the evening discount
from 5% to 10%, means editing two files that share no name or type. If only
`reservationManager.ts` is changed, new bookings and receipts use the new price while
`revenue()` keeps applying the old one, so the report's totals disagree with what customers
were charged. No test would catch it. `reporting.test.ts` checks the total against
`morning.priceCents + evening.priceCents` only in the case where both formulas agree. The
duplication also means bookings that were stored at an old price get repriced at the current
one whenever `revenue()` runs.

### Smell 2

**The smell.** Speculative generality / dead code: a read-through cache that is never
populated. The manager reads the cache but nothing ever writes to it, and nothing invalidates
it when bookings change.

**Classic or agent-specific.** Agent-specific: *phantom complexity*. The cache looks like
working machinery (TTL expiry, size-bounded eviction, a config type, `withTtl` and `disabled`
builders, `invalidate`, `size`) but nothing feeds it, so none of it can affect a result. It is
distinct from speculative over-abstraction because there is no abstraction layer here to defend.
The problem is that a whole mechanism *appears* to run and does not. `withTtl` and `disabled`
in `cacheConfig.ts` are also never called anywhere in `src/` or `tests/`.

**Cause (from the lecture).** Free volume. Generating a cache with expiry, eviction and config
builders costs the author nothing, so it got included. Every line of it still has to be
maintained and read by the next agent, and none of it does anything. The slide's last point
applies: a human could ship the same dead module, just more slowly.

**Where in the code.** [src/cache/queryCache.ts](src/cache/queryCache.ts) (`set`, `invalidate`
and `size` have no callers), [src/cache/cacheConfig.ts](src/cache/cacheConfig.ts), and the read
in [src/reservationManager.ts](src/reservationManager.ts) `listBookingsForRoom` (lines
117-124), which is the only call into the cache. `cache.get` always returns `undefined` today.

**The principle it violates.** YAGNI, along with keeping the code honest about what it does.
`listBookingsForRoom` reads as if it were cached when it never is.

**What it makes expensive.** The obvious next step is to "finish" the cache by adding a
`cache.set` after the storage read. That would introduce stale reads immediately.
`createBooking` and `cancelBooking` never call `invalidate`, so for up to 30 seconds
`listBookingsForRoom` and `formatDailySummary` would show a cancelled booking as confirmed
and miss new ones. Whoever does this has to discover that the write path is uninvolved. Until
then, every reader of `listBookingsForRoom` has to work out that the cache is inert, and the
manager carries a dependency it does not use.

### Smell 3

**The smell.** An extension mechanism built for a future that does not exist, while the one
seam that matters is missing. `notifierFactory.ts` provides a registry, a config type and a
`ChannelName` union for interchangeable channels, but the union has one member (`'email'`) and
`registeredChannels` has no callers. `ReservationManager` then builds its channel from a
module-level default and cannot be given one, unlike storage. `dispatchNotification` also
discards `NotificationResult.delivered`.

**Classic or agent-specific.** Agent-specific: *speculative over-abstraction*, with a classic
core. The registry, factory and config union are the speculative part: one implementation,
mutable module-level state, and a registration that runs as an import side effect, all so that
channels can be swapped. That is unrequested flexibility. The telltale is that the flexibility
does not reach the manager's constructor, which is where swapping would actually happen. The
inconsistency with storage, which *is* injected, is a smaller *convention drift* signal. The
underlying defect, a hard-wired dependency plus an ignored return value, is classic and humans
write it too, so this is the weakest agent-specific label of the three.

**Cause (from the lecture).** Underspecified request. "Send a confirmation" does not say how
general to be, so the gap was filled from habit: a factory, a registry and a config union for
channels that were never requested. The same gap left the constructor's notifier question open,
which is why the manager ended up hard-wired.

**Where in the code.** [src/reservationManager.ts](src/reservationManager.ts) the constructor
(line 40, `createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`) and `dispatchNotification`
(lines 215-218). Related: [src/notifications/notifierFactory.ts](src/notifications/notifierFactory.ts),
where `ChannelName` is the single literal `'email'`.

**The principle it violates.** Dependency inversion. The manager depends on a concrete
config and factory instead of receiving a `NotificationChannel`, even though it already
depends on the `StorageProvider` interface the right way. Also, the interface reports
`delivered`, and the caller throws that information away.

**What it makes expensive.** Testing the failure path is the first thing that breaks: there
is no way to hand the manager a channel that returns `delivered: false` or throws, so
"what happens when confirmation fails to send" cannot be tested at all. Today it would be
recorded in `notificationLog` as if it had gone out, for example
`email:alice@example.edu:Reservation confirmed`. Adding a second channel such as SMS means
editing the `ChannelName` union, registering a builder, and editing the manager's constructor
and default config, because there is no way to pass the channel in.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, the duplicated pricing rule. I chose it because it is
the only one of the three where the fix changes no behavior. Smell 2 (the cache) would mean
either deleting a feature or wiring it up, and Smell 3 (the notifier) would mean changing the
constructor's public shape. Smell 1 is also the one that can silently produce wrong numbers: a
price change made in one file leaves the other file's revenue report out of date.

**What changed.**
- New [src/pricing.ts](src/pricing.ts) holds the five pricing constants and one function,
  `calculatePrice(room, start, end)`. It applies the premium surcharge, then the long-booking
  discount, then the evening discount, in the same order and with the same rounding as before.
- [src/reservationManager.ts](src/reservationManager.ts): `calculatePrice` keeps its public
  signature and now just calls the shared function. The private `applyDiscounts` and the five
  local constants are gone.
- [src/reportGenerator.ts](src/reportGenerator.ts): `priceOf` now calls the shared function.
  The private `durationOf` and the five local constants are gone.
- Net effect: the pricing rules exist in one place. Changing the evening discount is now a
  one-line edit that both receipts and `revenue()` pick up.

**What I deliberately did not touch.** The scope line is: *move where the price is computed,
not what is computed or who computes it.*
- `revenue()` still recomputes each price instead of summing the stored `booking.priceCents`.
  Switching would be the better design, but it changes behavior: bookings priced under an old
  rule would report their old price rather than being repriced. That is a decision about
  reporting semantics, so it belongs in its own change.
- The overlap logic (`hasConflict`, `isSlotFree`, `overlapsWindow`) is also duplicated, but it
  is a different smell in a different part of the code.
- `ReservationManager.calculatePrice` stays as a thin public wrapper, so no caller changes.
- Smells 2 and 3 are untouched. They are Milestone 3 proposals.

I stopped there because everything past that line either alters observable output or belongs
to a different smell. Each of those changes should get its own review.

**How you know behavior is preserved.** After `npm ci`, `npm test` passes all 39 tests
(3 files) and `npm run typecheck` reports no errors. No test files were edited
(`git status` shows only `SMELLS.md`, `src/reservationManager.ts`, `src/reportGenerator.ts`
and the new `src/pricing.ts`). I ran the suite before and after; the baseline run failed only
because dependencies were not installed. What the suite covers: `booking.test.ts` pins exact
cents for the base rate (12000), the long-booking discount (16200), the premium surcharge
(18400) and the evening discount (11400), and `reporting.test.ts` checks revenue totals for a
standard room. What it would not catch: no test combines the rules (premium, 3 hours or more,
and evening together), so a change in the order of the multiplications would go unnoticed,
and the report's pricing is only exercised for a standard room. I kept the order and the
rounding identical to the original by hand. The premium, long-booking and evening
multipliers were applied in the same order in both original copies, so extracting them did
not require choosing between two versions.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

*For Smell 2, the cache that is never filled.*

**The problem.** `ReservationManager.listBookingsForRoom` reads from a `QueryCache` that nothing
ever writes to, so it is inert. If someone "finishes" it by adding a `cache.set`, the manager
becomes responsible for invalidating on every write path (`createBooking`, `cancelBooking`, and
any future one), and today none of them do.

**The decomposition.**
- Step 1, the smallest change: delete the cache from the manager. Drop the `cache` field, its
  constructor line and the read in `listBookingsForRoom`, which becomes a direct
  `storage.findByRoom` call. Delete `cache/` if no caller appears.
- Step 2, only if caching is ever needed: a `CachingStorageProvider` that implements
  `StorageProvider` and wraps another provider. It is the only place that sees every `save`,
  `update` and `clear`, so it owns the rule "any write to a room drops that room's cached
  reads". `QueryCache` keeps owning TTL and eviction and nothing else. The manager and
  `ReportGenerator` stay unaware, because they already depend on the interface, not on a
  concrete class.

**One cost.** Cached results would be shared references. `InMemoryStorageProvider` returns
copies so callers cannot mutate stored bookings, but a cache that hands back the same
`Booking[]` on every hit lets one caller corrupt what the next caller sees. The decorator would
have to copy on the way out, which costs back part of what the cache was meant to save. Step 1
also permanently discards TTL and eviction code that took effort to write; it is recoverable
from git but not free.

### Proposal B (not coded)

*For Smell 3, the hard-wired notifier and the ignored `delivered` flag.*

**The problem.** The manager builds its own notifier from `DEFAULT_NOTIFIER_CONFIG`, so tests and
callers cannot supply one, and `dispatchNotification` records every send as if it succeeded.

**The decomposition.**
- `ReservationManager`'s constructor takes the channel as a second optional parameter, defaulting
  to `createNotificationChannel(DEFAULT_NOTIFIER_CONFIG)`. Existing callers and `newService()`
  in the fixtures keep working. `notifierFactory` stays what it is: the startup wiring that
  turns config into a channel.
- `NotificationChannel` stays the boundary. It reports what happened (`delivered`, `message`) and
  makes no policy decisions.
- The manager owns the policy, which today does not exist: what to do when `delivered` is false
  or `send` throws. The log entry would say so, for example `email:alice@example.edu:Reservation
  confirmed:failed`.
- Optionally, extract the log and this policy into a small `NotificationDispatcher`, so the
  manager stops owning `notificationLog`. I would do that as a second step, not the first.

**One cost.** It forces a decision that is currently hidden. By the time `dispatchNotification`
runs, the booking is already saved. If a failed send should not fail the booking, the
failure is only visible to someone who reads `recentNotifications()`. If it should fail, the
caller gets an exception for a booking that exists. Neither is free, and the current code
avoids the question by not looking. Also, no test currently touches notifications, so nothing
would catch a mistake in this change until tests are written first.

### The thing that looks smelly but is fine

**What it is.** [src/validation.ts](src/validation.ts) `validateReservationRequest`. It is a
50-line function with about a dozen `if` branches, and it reads like a candidate for
"long method".

**Why it is fine.** Length is not the problem; coupling is, and this function has almost none.
- It is pure: two arguments in, one `ValidationResult` out. It reads no clock, storage or
  other state, so it needs no setup to test.
- The checks are flat, early-return and independent. No branch depends on a variable set by
  another, so nothing needs to be held in your head across the function. Reading top to bottom
  is the whole design, and the comments mark the groups (shape, times, duration, capacity,
  building rules).
- The order is the contract. It returns the first problem so callers get one clear reason,
  and `validation.test.ts` pins every message through a table of rejections, so a reorder or a
  dropped rule fails a test.
- Its constants are named and used only here, so there is no second copy to drift, unlike the
  pricing constants in Smell 1.
- Splitting it into one function per rule would mostly add indirection: a dozen tiny functions
  and a list to keep in the right order, for no gain in testability.

**What would flip your verdict.** Any of these:
- Rules that vary per building or per room (for example, different opening hours), so the
  constants stop being constants and the function needs a config or policy object.
- A need to report *all* failures at once instead of the first, which breaks the early-return
  structure.
- Rules that need outside state, such as the user's role or existing bookings, which would
  make it impure.
- Any other module starting to duplicate one of these rules. That would make it Smell 1 all
  over again.
