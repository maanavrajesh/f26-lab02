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

`everyMinuteOfTheDayIsExactlyOneOfBookedOrFree` in `AvailabilityProperties.java`, built on the
provided `scenarios()` generator. It walks every minute of the business day and asserts
`booked != reportedFree`: exactly one, never both and never neither.

It failed on the first run. jqwik shrank the counterexample to a one-minute day with nothing
booked:

    Scenario[dayStart=0, dayEnd=1, bookings=[]]

`freeSlots` returns an empty list for it. The provided no-overlap property passes on the same
sample because it only inspects slots that came back — with none returned, its loop body never
runs. Mine fails because minute 0 is neither booked nor reported free.

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

Actions are disabled by default on a fresh fork and GitHub does not backfill missed runs, so
the Milestone 1 commit (`3f428ed`) has no run of its own. `170c2c2` is the same broken
calculator and the same failing property, pushed once Actions were on.

## Tools used

Claude Code (Claude Opus 5) — setup verification, drafting the property, diagnosing and fixing
the bug, the suite audit, and this write-up.
