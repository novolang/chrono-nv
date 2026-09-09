# chrono-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The half of a date library that asks the machine. `calendar-nv` does
the arithmetic and has no clock in it; this one has the clock, the
host's UTC offset, a monotonic reading, deadlines and a stopwatch — and
the bridge across to `std.time`'s own types, so a program can hold both
without unpacking integers at every seam.

```
novo pkg add chrono-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use clock
use zone
use iso8601

fn stamp() -> Str [time, io]
    let at = match zone.local()
        Ok(o)  => o
        Err(_) => zone.utc()
    iso8601.format_rfc3339(clock.now(at), at.minutes)
```

The fallback is written out because that is the honest shape: a host
can fail to have a zone, and a program that logs anyway should log in
UTC and be able to say so.

## The load-bearing interface

Not a type — the **effect rows**, which are three different answers to
"what did this call have to touch":

```novo
pub fn between(earlier: Tick, later: Tick) -> TimeDelta          // nothing
pub fn offset_at(when: CivilDateTime) -> Result<Offset, …> [io]  // the environment
pub fn now(at: Offset) -> CivilDateTime [time]                   // the clock
pub fn local() -> Result<Offset, …> [time, io]                   // both
```

`[io]` for the zone because SPEC § 5.1 puts the environment under it,
and `[time]` for the clock. Every convenience in this package is one of
those rows plus a call, and the split is deliberate: a caller who
already has an instant should not be charged `[time]` to translate it,
and a caller in a sandbox that grants one label and not the other can
see which calls they still have before they write them.

The same split is why the pure functions are here rather than folded
away — `mono.between`, `timer.deadline_at`, `clock.from_unix`,
everything in `bridge`. They need this package's `Offset` or `Tick`, so
they cannot live in `calendar-nv`; they touch nothing, so they declare
nothing.

## Two clocks, and they are not interchangeable

| | `clock` | `mono` |
| --- | --- | --- |
| answers | what does the calendar say | how much time has passed |
| can go backwards | **yes** — NTP, a person, a VM snapshot | no |
| has a calendar meaning | yes | **no** |
| use it for | timestamps, dates, log lines | durations, deadlines, benchmarks |

They are separate modules rather than separate functions in one,
because the mistake this prevents is reaching for whatever is nearest
and called `now`. A `Tick` has no conversion to a `CivilDateTime` and
never will; a `Deadline` is built on `Tick` and nothing else, because a
deadline on the wall clock expires at the wrong moment on the day the
clocks change.

## The layer, and why

`host` — every effect in the calendar/clock split lives here, which is
the point of the split. `calendar-nv` is `core` and depends on nothing;
this package depends on it, which is the direction the layer order
allows and the one that keeps the arithmetic testable with no clock in
the room.

There is no `[fs]` and no `[net]`: the zone comes from the environment,
which is `[io]`, and nothing here opens a file or a socket. Reading a
zone database from disk would be `[fs]`, and that is `tz-nv`'s row on
the plan, not this one's.

## The names, and the one that was taken

`Instant` is a standard-library struct name and redeclaring one is a
compile error, so the monotonic reading is **`Tick`** — which is also
the name `std.time`'s own monotonic free functions use
(`core.time.ticks()`), so it reads consistently with the layer
underneath. `calendar-nv`'s README has the same table for `Date`,
`Duration` and the rest.

## The reference implementation

Rust's `chrono`, the half that was left out of `calendar-nv`: `Utc::now`,
`Local::now`, `Offset`, `DateTime<Tz>`'s construction from a timestamp.
`std::time::Instant` for the monotonic type's contract — an opaque
reading with no epoch, comparable only within a process. Python's
`datetime.now(tz)` for the insistence that a wall-clock reading names
the zone it was taken in.

Deliberately not ported, and where it went instead:

- **A zone database.** `Offset` is a number of minutes; the rules that
  produce one for a named zone and an instant are `tz-nv`'s.
- **`DateTime<Tz>` as one type.** Here a timestamp is a
  `CivilDateTime` and an `Offset`, kept apart, because that is what
  lets the arithmetic live in a `core` package.
- **Async timers.** `Deadline` is a value a loop asks; the executor
  half is `timer-nv`'s row on the plan.
- **`chrono::Duration`** — it is `calendar-nv`'s `TimeDelta`, and this
  package converts to and from `std.time.Duration` in `bridge`.

## Status

Every function is `todo()`. `novo test` runs the API suite, and every
assertion in it reaches `not implemented: chrono-nv.<fn>` — the
expected result until the bodies land. Run it with `--isolate` for one
verdict per test naming the function it stopped at.

| module | functions | rows | implemented |
| --- | --- | --- | --- |
| `chronoerror` | 1 | none | no |
| `zone` | 4 | `[io]`, `[time, io]` | no |
| `clock` | 10 | `[time]`, `[time, io]` | no |
| `mono` | 4 | `[time]` | no |
| `timer` | 7 | `[time]` | no |
| `bridge` | 5 | none | no |
