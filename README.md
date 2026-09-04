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

What I did for each milestone, and what to look at during the TA check-off.

## Setup verification

Before starting I checked every step in `SETUP.md` on this machine:

- JDK 21.0.12.1 (Temurin) and Maven 3.9.9, both on the path.
- `mvn test` on the untouched starter: BUILD SUCCESS, 7 tests (6 example-based plus the one
  provided property).
- A clean build against an empty local Maven repo also passes, so jqwik and JaCoCo resolve
  from Maven Central with nothing pre-cached.
- The coverage report generates at `target/site/jacoco/index.html`, where
  `AvailabilityCalculator` shows **100% instruction and 100% branch coverage** — which is the
  point the lab is making: the bug below survives a fully covered, fully green suite.

## Milestone 1 — specify the invariant as a property

### The property

`everyMinuteOfTheDayIsExactlyOneOfBookedOrFree`, added to
`src/test/java/edu/cmu/cs214/availability/AvailabilityProperties.java` along with a small
`covers(intervals, minute)` helper. It reuses the provided `scenarios()` generator, so it sees
the same unsorted, overlapping, out-of-hours bookings the example property does.

It walks the business day one minute at a time and asserts, for every minute in
`[dayStart, dayEnd)`:

    booked != reportedFree

That inequality is the "exactly one" claim from `ARCHITECTURE.md`. It fails if a minute is
**both** covered by a booking and returned as free, and it fails if a minute is **neither** —
the half the provided property cannot see.

### The failing sample

`mvn test` fails with 8 tests run, 1 failure. jqwik reported:

    Shrunk Sample (3 steps)
    -----------------------
      arg0: Scenario[dayStart=0, dayEnd=1, bookings=[]]

    Original Sample
    ---------------
      arg0: Scenario[dayStart=23, dayEnd=1185, bookings=[TimeInterval[start=135, end=359]]]

(Re-runs print this under `Sample` instead of `Shrunk Sample`, because jqwik replays the last
failing sample first from `.jqwik-database`. The seed for the original run was
`928765526283672927`.)

### What the calculator returns, and why the two properties disagree

For the shrunk sample — a one-minute day `[0, 1)` with no bookings — `freeSlots` returns the
**empty list**. The whole day is free, and it reports none of it.

- **The provided "no overlap" property still passes.** It loops over the slots that came back
  and checks each one against the bookings. Nothing came back, so the loop body never runs and
  the property is vacuously true. It can only catch a slot that *is* returned and shouldn't be;
  it is structurally blind to a slot that is missing.
- **My property fails.** Minute 0 lies in `[dayStart, dayEnd)`. No booking covers it
  (`booked = false`) and no returned slot covers it (`reportedFree = false`), so the minute is
  "neither" and the assertion trips:

      minute 0 is neither booked nor reported free; bookings=[], free=[]

The unshrunk sample shows the same bug with more meat on it: for the day `[23, 1185)` with one
booking `[135, 359)`, the calculator returns only `[TimeInterval[start=23, end=135]]` and
silently drops the free stretch `[359, 1185)` — over 13 hours of availability.

### The bug

In `AvailabilityCalculator.freeSlots`, the loop emits a gap only *before* each booking's start
and then returns. It never emits the final gap from `cursor` to `dayEnd`, so any free time
after the last booking is lost. With no bookings at all, `cursor` never moves off `dayStart`
and the entire day is dropped.

The generated example suite misses this because every one of its bookings ends exactly at
`DAY_END`, which leaves `cursor == dayEnd` and no tail gap to lose — and no test ever passes an
empty booking list. That is how it reaches 100% coverage while being green on broken code.

### For the TA

- The property: `everyMinuteOfTheDayIsExactlyOneOfBookedOrFree` in `AvailabilityProperties.java`.
- The failing sample, and the two questions above: which minute breaks, what `freeSlots` returns
  for it, and why the provided property is blind to it.
- Commit `3f428ed` adds the property **on its own, before any fix**, and the CI run it
  triggers on the Actions tab is red. This README landed in a later commit so the milestone
  commit stays clean.

## Milestone 2 — fix the bug

### A note on the Actions tab

GitHub disables Actions on a fresh fork, and I had not enabled them before the Milestone 1
push, so that push produced no CI run at all (the workflow was registered but
`actions/runs` reported `total_count: 0`). Enabling Actions does not backfill missed runs.
The red run on the tab is therefore the commit below, which was pushed *after* Actions was
enabled and *before* any fix — it runs the same broken calculator and the same failing
property as the Milestone 1 commit.

### Diagnosis

`freeSlots` sorts and clips the bookings, then sweeps them with a `cursor`, emitting a gap
only *before* each booking's start. When the loop ends it returns immediately, so the final
stretch from `cursor` to `dayEnd` is never emitted:

- With bookings, all free time after the **last** booking is dropped.
- With no bookings at all, `cursor` never moves off `dayStart` and the **entire day** is
  dropped — which is the shrunk sample jqwik found, `Scenario[dayStart=0, dayEnd=1, bookings=[]]`.

The fix is to emit that trailing gap after the loop, guarded by `cursor < dayEnd` so that a
fully booked day still yields nothing and no empty interval is ever constructed (which
`TimeInterval` would reject). The property is not touched.

### The fix

`AvailabilityCalculator.freeSlots` now emits the trailing gap after the sweep loop:

    if (cursor < dayEnd) {
        free.add(new TimeInterval(cursor, dayEnd));
    }

Three lines, in the calculator. The property was not weakened or changed in any way.

### Verification

`mvn test` is green: **8 tests, 0 failures** — the 6 example tests, the provided no-overlap
property, and `everyMinuteOfTheDayIsExactlyOneOfBookedOrFree` at 1000 tries / 1000 checks.
Verified twice: once replaying the saved failing sample from `.jqwik-database` (jqwik's
`SAMPLE_FIRST` mode), and once with that database deleted so the run generated entirely fresh
inputs.

### For the TA

**One sentence:** `freeSlots` never emitted the final gap between the last booking and the end
of the business day, so all free time after the last booking — and the entire day when there
were no bookings — was silently dropped.

The two runs to show side by side on the Actions tab:

| Commit | Contains | CI |
|---|---|---|
| `170c2c2` | the failing property, no fix | red |
| the commit below it | the three-line fix | green |

## Milestone 3 — not started yet
