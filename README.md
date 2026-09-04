# Lab 2 Starter: Availability Calculator

A small reservation component. Given a room's bookings and the day's business hours,
`AvailabilityCalculator.freeSlots` computes when the room is free. It is the code you
work in for Lab 2.

It ships with a generated test suite that passes, and a property-based test harness
(jqwik) with one example property. Everything is green. Your job in Lab 2 is to decide
whether green actually means correct.

**Read `ARCHITECTURE.md` before the code.**

## Build and test

```
mvn test
```

`mvn test` runs both files, the ordinary example-based tests (`AvailabilityCalculatorTest`)
and the property-based tests (`AvailabilityProperties`). A code-coverage report is written
to `target/site/jacoco/index.html`.

## Continuous integration

This repository has CI configured in `.github/workflows/ci.yml`. GitHub disables workflows on a
fresh fork, so enable them once on your fork (the handout shows where). After that, every
push runs `mvn test`. You will watch the gate go red when your new property finds the bug, then
green once you fix it.

## Where things are

- Component: `src/main/java/edu/cmu/cs214/availability/`
- Example-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityCalculatorTest.java`
- Property-based tests: `src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java`
- Setup: `SETUP.md`

See the Lab 2 handout on the course page for the three milestones you show a TA.

---

# Lab 2 write-up

## Milestone 1 — the property

Added to `AvailabilityProperties.java`, built on the provided `scenarios()` generator so it
sees the same unsorted, overlapping, out-of-hours bookings the example property does:

```java
@Property
void everyMinuteOfTheDayIsExactlyOneOfBookedOrFree(@ForAll("scenarios") Scenario s) {
    List<TimeInterval> free = calc.freeSlots(s.dayStart(), s.dayEnd(), s.bookings());
    for (int minute = s.dayStart(); minute < s.dayEnd(); minute++) {
        boolean booked = covers(s.bookings(), minute);
        boolean reportedFree = covers(free, minute);
        int m = minute;
        assertTrue(booked != reportedFree,
            () -> "minute " + m + " is "
                + (booked ? "both booked and reported free" : "neither booked nor reported free")
                + "; bookings=" + s.bookings() + ", free=" + free);
    }
}
```

`covers(intervals, minute)` is a small helper asking whether any interval in the list contains
that minute. The assertion `booked != reportedFree` is the "exactly one" claim: a minute may
not be both booked and reported free, and it may not be neither.

### The smallest failing sample

It failed on the first run. jqwik shrank the counterexample to a one-minute day with nothing
booked — printed under `Shrunk Sample` on that run, and under `Sample` on re-runs, since jqwik
replays the last failing sample from `.jqwik-database` first:

    Scenario[dayStart=0, dayEnd=1, bookings=[]]

**What the calculator returns for it.** An empty list. There are no bookings to sweep, so
`cursor` never leaves `dayStart` and nothing is ever emitted. The whole day is free and none of
it is reported.

**Why the provided no-overlap property still passes.** It iterates the slots that came back and
checks each one against the bookings. Nothing came back, so the loop body never executes and
the property is vacuously true. It can only catch a slot that *is* returned and should not be;
it is structurally blind to a slot that is missing.

**Why mine fails.** Minute 0 lies in `[dayStart, dayEnd)`. No booking covers it and no returned
slot covers it, so `booked` and `reportedFree` are both false, the minute is "neither", and the
assertion trips:

    minute 0 is neither booked nor reported free; bookings=[], free=[]

The unshrunk sample shows the same bug at realistic scale: for the day `[23, 1185)` with one
booking `[135, 359)`, the calculator returns only `[23, 135)` and drops the free stretch
`[359, 1185)`.

Pushed on its own, before any fix.

## Milestone 2 — the fix

`freeSlots` emitted a gap only *before* each booking and returned as soon as the sweep loop
ended, so free time after the last booking was dropped, and with no bookings at all the entire
day was. The fix emits the trailing gap:

    if (cursor < dayEnd) {
        free.add(new TimeInterval(cursor, dayEnd));
    }

The guard keeps a fully booked day empty and avoids constructing an empty interval. The
property was not weakened.

## Milestone 3 — auditing the generated suite

`AvailabilityCalculatorTest` held 100% instruction and branch coverage on the calculator while
the bug was live. Three concrete reasons:

1. **Every booking runs to the close of business.** Five of the six tests end their last
   booking exactly at `DAY_END`, leaving `cursor == dayEnd` when the loop exits, so there is
   never a trailing gap to lose. *Controllability gap* — the input that exposes the bug was
   never driven.

2. **No test passes an empty booking list.** Every test supplies at least one booking, so the
   worst case — an empty day, where the whole day is free and the calculator returns nothing —
   is never reached. It is exactly the sample jqwik shrank to. *Controllability gap.*

3. **`returnedSlotsNeverOverlapABooking` can only see what was returned.** Its booking
   `[600, 660)` does *not* run to `DAY_END`, so this test genuinely executes the buggy path and
   the free stretch `[660, 1020)` is silently dropped. It still passes: the assertion iterates
   the returned list checking each slot for overlap, and the missing slot is not in the list to
   be iterated. *Observability gap* — it ran the bug and could not see it.

**Why the coverage number did not help.** Coverage records which lines and branches executed,
not whether any assertion could tell a right answer from a wrong one. Weakness 3 is the two
coming apart: the bug-exposing input ran, and the test passed anyway. And the defect is
*absent* code — there was no statement emitting the trailing gap, so there was no line for
JaCoCo to mark red. Every line that existed ran; the bug was in the line nobody wrote.

## Final result

`mvn test` is green at 8 tests, 0 failures: the 6 example tests, the provided property, and the
new one at 1000 tries. Verified both replaying the saved failing sample and with
`.jqwik-database` deleted for fresh inputs.

CI on the fork went red then green:

| Commit | Contains | CI |
|---|---|---|
| `170c2c2` | the failing property, no fix | red |
| `aed1d6e` | the three-line fix | green |

The tip of `main` runs green too, so the finished state of the repo is verified by CI and not
just locally.

Actions are disabled by default on a fresh fork and GitHub does not backfill missed runs, so
the Milestone 1 commit (`3f428ed`) has no run of its own. `170c2c2` is the same broken
calculator and the same failing property, pushed once Actions were on.

## Tools used

Claude Code (Claude Opus 5) — setup verification, drafting the property, diagnosing and fixing
the bug, the suite audit, and this write-up.
