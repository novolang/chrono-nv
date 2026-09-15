# chrono-nv

A **wall clock** answers what the calendar says. A **monotonic clock**
answers how much time has passed. This package is the half of a date
library that asks a machine for either: the wall clock, today, the
host's offset from UTC, a monotonic reading, deadlines and a stopwatch.
The arithmetic it answers in belongs to
[calendar-nv](https://novo-lang.org/packages/calendar-nv), which has no
clock in it and which this package depends on. The reference
implementation is Rust's
[chrono](https://docs.rs/chrono), the half that `calendar-nv` left out:
`Utc::now`, `Local::now`, `Offset` and the construction of a timestamp
from a Unix second.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A machine has two clocks and they answer different questions. The table
is the whole reason this package has two modules rather than two
functions in one.

| | `clock` | `mono` |
| --- | --- | --- |
| Answers | what does the calendar say | how much time has passed |
| Can go backwards | yes | no |
| Has a calendar meaning | yes | no |
| Reach for it to | stamp a log line, name a file, show a date | time an operation, set a timeout, run a benchmark |

The wall clock goes backwards whenever something sets it: the Network
Time Protocol steps it, a person corrects it, a virtual machine resumes
from a snapshot. The monotonic clock is a counter from an unspecified
origin that only moves forward. A reading of it is a **`Tick`**, and a
`Tick` has no calendar meaning: it is comparable with other readings
from the same process and with nothing else. There is no conversion
from a `Tick` to a date, and there will not be one.

An **offset** is a number of minutes east of UTC. `120` is `+02:00`,
`-330` is `-05:30` and `0` is `Z`. An offset is not a time zone.
`+02:00` is a number; `Europe/Copenhagen` is a set of political rules
that change, and this package carries none of them.
[tz-nv](https://novo-lang.org/packages/tz-nv) is the package that does.

Every function here declares what it had to touch, and there are four
answers. The declaration is checked by the compiler, so a caller can
see what a call costs before writing it.

| Declared effects | What the function touched | Example |
| --- | --- | --- |
| none | nothing; both values are the caller's | `mono.between`, `clock.from_unix` |
| `[time]` | a clock | `clock.now`, `mono.read` |
| `[io]` | the environment, where the host's zone is read from | `zone.offset_at` |
| `[time, io]` | both | `zone.local`, `clock.now_local` |

SPEC section 5.1 puts the environment under `[io]`. A caller who
already holds an instant is not charged for a clock to translate it,
and a caller running where one of the two is granted and the other is
not can tell which calls are still available.

These are the numbers the package is measured against.

| Quantity | Value |
| --- | --- |
| Unit of an `Offset` | minutes east of UTC |
| Offset range | -1080 to 1080 minutes |
| That range in hours | eighteen either way |
| Unix epoch | 1970-01-01T00:00:00Z |
| Resolution of a `Tick` | nanoseconds |
| Resolution of `std.time.Mono` | microseconds |
| Resolution of `std.time.Duration` | microseconds |
| Resolution of `calendar-nv`'s `TimeDelta` | nanoseconds |

## Install

```
novo pkg add chrono-nv
```

## Example

```novo
use clock
use iso8601
use mono
use span
use timer
use zone

// One log line, stamped with the time and the offset it was read at.
fn stamp(message: Str) -> Str [time, io]
    // Ask the host for its offset from UTC. A host with no zone
    // configured has none to give, so fall back to UTC and carry on.
    let at = match zone.local()
        Ok(o)  => o
        Err(_) => zone.utc()

    // Read the wall clock at that offset and write the reading as an
    // RFC 3339 timestamp, such as 2026-09-15T14:05:00+02:00.
    "${iso8601.format_rfc3339(clock.now(at), at.minutes)} ${message}"

fn main() [io, time]
    println(stamp("started"))

    // The other clock. A stopwatch measures how much time passed, so
    // it reads the monotonic clock, which cannot go backwards.
    let t = timer.start("work")
    mono.sleep(span.milliseconds(50))

    // The measurement is a TimeDelta, the same type a calendar
    // difference gives, so the same functions format and compare it.
    println("${span.as_seconds(timer.stop(t))}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: chrono-nv.<fn>` panic. The tests are the
specification the implementation will have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `zone` | `Offset`, a number of minutes east of UTC, and three ways to get one: UTC, a number the caller supplies, and the host's own. |
| `clock` | The wall clock: the date-time now at an offset, today's date, the Unix second and nanosecond, and the conversions between a Unix instant and a civil date-time. |
| `mono` | The monotonic clock: a reading, the time since a reading, the length between two readings, and a blocking sleep. |
| `timer` | `Deadline`, a point on the monotonic clock a loop can ask about, and `Timer`, a stopwatch that carries the name of what it is timing. |
| `bridge` | The conversions to and from the standard library's `Zoned`, `Duration` and `Mono`. |
| `chronoerror` | The five reasons a clock, a zone or a conversion could not answer, including the calendar's own reason carried through unchanged. |

## How to choose an entry point

**Choose the clock by the question, not by which `now` is nearest.**
A date, a timestamp and a log line come from `clock`. A timeout, a
duration and a benchmark come from `mono`. A deadline taken from the
wall clock expires at the wrong moment on the day the clocks change.

There are three ways to get an offset, and they cost different things.

**`zone.utc()` touches nothing.** It is the right answer for a program
that has decided to log in UTC, and the fallback when the host has no
zone.

**`zone.offset_at(when)` reads the environment and not the clock.** The
caller supplies the instant, so the answer is the same on every run
with the same argument. This is the call to make from a test, and the
call available where `[io]` is granted and `[time]` is not.

**`zone.local()` reads both.** It answers the host's offset right now,
which is what a program printing something for a person wants.

The wall-clock calls follow the same three shapes. `clock.now_utc()`
assumes UTC and says so in its name. `clock.now(at)` takes the offset
the caller decided on. `clock.now_local()` asks the host, and costs
`[time, io]` for doing so.

**`timer.deadline_at` is the constructor a test uses.** It takes a
reading the caller already has, so whether the deadline has expired is
decidable without waiting. `timer.deadline_in` takes a length and reads
the clock itself.

## The rules a user needs

1. **There is no bare `now()`.** Every wall-clock call either takes an
   offset or names the one it assumes. A civil date-time with no zone
   attached and no zone named is the value that produces
   off-by-an-hour bugs in the week after a clock change.
2. **A `Tick` has no calendar meaning and no conversion to a date.**
   It is a reading from an unspecified origin, comparable only with
   other readings from the same process. Two `Tick`s from two runs of a
   program say nothing about each other.
3. **A `Deadline` is built on a `Tick` and nothing else.** There is no
   constructor here that takes a civil date-time. A caller who means
   "at 09:00 tomorrow" is scheduling rather than timing out, and wants
   a difference from `clock.now(at)` converted into a length first.
4. **An offset is not a zone.** `zone.fixed` and `zone.local` answer a
   number of minutes. The rules that produce that number for a named
   zone at a given instant are tz-nv's.
5. **An offset outside eighteen hours either way is refused.**
   `zone.fixed(1081)` is `OffsetOutOfRange(1081)`. RFC 3339 section 4.2
   defines the local offset, and no zone database goes further than
   this range.
6. **`zone.offset_at` answers the offset in effect before a
   daylight-saving transition.** A civil time inside a transition
   either happens twice or does not happen at all. This function takes
   the same reading `mktime` takes with `tm_isdst = 0` in both cases,
   which is what makes it total. A caller who needs to know that the
   time was ambiguous needs a zone database.
7. **`today` takes an offset, and the argument matters most around
   midnight.** Two callers at different offsets are on different days
   for those hours. A log rotation on UTC that a person reads in local
   dates is the bug this argument prevents.
8. **`mono.between` touches nothing and `mono.elapsed` reads the
   clock.** `between` takes two readings the caller already has.
   `elapsed` takes one and reads the second itself.
9. **`mono.elapsed` is never negative; `mono.between` is negative when
   the arguments are the wrong way round.** The clock does not go
   backwards, so a negative difference is the caller's mistake.
10. **`mono.sleep` waits at least the length asked for.** A scheduler
    may return late and no promise is made that it will not. A zero or
    negative length returns immediately rather than failing.
11. **`timer.remaining` goes negative once the deadline has passed.**
    How late a program is is a thing it logs, and a clamp to zero
    throws that away. A transport that wants a poll timeout treats a
    negative length as "do not wait".
12. **`timer.lap` hands back a new `Timer`.** The stopwatch is threaded
    rather than mutated, so a caller keeps the value the call returns.
    The first lap measures from the start.
13. **A `TimeDelta` converted to `std.time.Duration` loses its
    nanosecond remainder.** `bridge.to_std_duration` truncates toward
    negative infinity, matching `span.as_seconds`, and answers
    `Overflow` when the length does not fit an `Int` of microseconds.
    The other three conversions are exact.
14. **There is no conversion from a `Tick` back to `std.time.Mono`.**
    A `Tick` may carry nanoseconds a `Mono` cannot hold. Convert the
    `TimeDelta` instead.
15. **`clock.unix_nanos` answers two `Int`s rather than a `Float`.**
    Fifty-two bits of mantissa resolves to about 0.2 microseconds
    around the present day, so a `Float` timestamp does not round-trip.
    `std.time.now()` is the `Float` surface and stays available.
16. **A failure that carries a `CalError` is about the value, and the
    others are about the machine.** `chronoerror.calendar_cause`
    answers which. A `CalError` means no retry helps. `NoLocalZone` and
    `ClockUnavailable` are about the environment, and a program can
    fall back or ask again. The `Error` trait that
    `Result<T, ChronoError>` requires is SPEC section 3.4.

## What is not included

- **A zone database.** The IANA rules that turn a zone name and an
  instant into an offset are
  [tz-nv](https://novo-lang.org/packages/tz-nv)'s.
- **A single zone-aware timestamp type.** Rust's `DateTime<Tz>` is one
  value; here a timestamp is a `CivilDateTime` and an `Offset`, kept
  apart. That is what lets the arithmetic live in a package with no
  effects in it.
- **Asynchronous timers and callbacks.** A `Deadline` is a value a loop
  asks about. The wheel a loop polls, with many deadlines in it, is
  [timer-nv](https://novo-lang.org/packages/timer-nv).
- **A file and a socket.** No function here declares `[fs]` or `[net]`.
  The host's zone comes from the environment, which is `[io]`. Reading
  a zone database off disk is `[fs]` and is tz-nv's.
- **Printing.** `timer.stop` answers a measurement and there is no
  variant of it that writes to a console. A library returns the number
  and the program decides where it appears.
- **Leap seconds.** The Unix second this package counts in has none.
  calendar-nv's RFC 3339 parser is where a `:60` in text is handled.

## Related packages

- [calendar-nv](https://novo-lang.org/packages/calendar-nv) is the
  other half: civil dates, times and lengths of time, with no clock and
  no zone. This package depends on it, answers in its types, and adds
  the effects it does not have.
- [tz-nv](https://novo-lang.org/packages/tz-nv) is the IANA time zone
  database as data. It produces the `Offset` this package takes as an
  argument, for a named zone at a given instant.
- [timer-nv](https://novo-lang.org/packages/timer-nv) is many deadlines
  in one wheel that a loop polls, with lateness policies, a token
  bucket and jitter. This package has one deadline at a time.
- [ntp-nv](https://novo-lang.org/packages/ntp-nv) asks a server how far
  this machine's clock is from a reference. It answers an offset in
  microseconds, which is what a program adds to a reading from here.
- `std.time` in the standard library is the machine's own clocks and
  its own types: `Mono`, `Instant`, `Duration` and `Zoned`, with RFC
  3339 formatting. Its `Tz` is a fixed number of minutes east of UTC.
  There is no zone-name lookup, no daylight saving and no historical
  offset table, so `Tz.parse("Europe/Copenhagen")` is `None`. That is a
  decision rather than a gap, written up in `docs/stdlib/time.md` under
  "Timezone data", and tz-nv is the package that carries the database.
  `bridge` converts between `std.time`'s types and this package's, in
  both directions where the conversion is exact.

## Tests

```bash
novo test tests/clock_tests.nv       #  9 tests: the wall clock and the zone
novo test tests/mono_tests.nv        # 11 tests: the monotonic half and the bridge
```

The reference implementations are Rust's `chrono` for the wall-clock
half, `std::time::Instant` for the monotonic type's contract, and
Python's `datetime.now(tz)` for the rule that a wall-clock reading
names the offset it was taken at.

A clock is hard to test, so the suite asserts what is true of every
reading rather than a value that would only be right for a minute: that
two readings are ordered, that an elapsed length is never negative,
that the offset argument changes the answer by exactly the offset. The
exact assertions are all on the functions that touch nothing:
`clock.from_unix` and `clock.to_unix` round-trip, `mono.between` is a
subtraction of the caller's own readings, and `timer.deadline_at` makes
a deadline whose expiry is decidable without waiting. The suite also
checks that `zone.utc()` is zero, that `zone.fixed` refuses past
eighteen hours, and that the conversions in `bridge` are exact in the
three directions that claim to be.

The tests compile today and fail at run, each on the
`not implemented: chrono-nv.<fn>` panic that is its body. That is the
expected state of an interface release. Run `novo test --isolate` for
one verdict per test, naming the function it stopped at. They turn
green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `zone.Offset`, `mono.Tick`, `timer.Deadline`, `timer.Timer` | declared |
| `chronoerror.ChronoError` | declared |
| `zone.utc`, `.fixed`, `.offset_at`, `.local` | no |
| `clock.now_utc`, `.now`, `.now_local` | no |
| `clock.today_utc`, `.today`, `.today_local` | no |
| `clock.unix_seconds`, `.unix_nanos` | no |
| `clock.from_unix`, `.to_unix` | no |
| `mono.read`, `.elapsed`, `.between`, `.sleep` | no |
| `timer.deadline_in`, `.deadline_at`, `.expired`, `.remaining` | no |
| `timer.start`, `.lap`, `.stop` | no |
| `bridge.from_zoned`, `.to_zoned` | no |
| `bridge.from_std_duration`, `.to_std_duration`, `.from_mono` | no |
| `chronoerror.calendar_cause`, `ChronoError.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
