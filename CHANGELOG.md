# Changelog

All notable changes to chrono-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-10

The **implementation**.  Every body the 0.0.x interface published is
written, `novo test --isolate` is green over three suites, and every
`pub fn` answers the value or the refusal its doc comment promises.

- **`ChronoError.Overflow("…").message()` binds the value first.**  A
  trait method called straight on a qualified constructor's temporary
  strands the string it answers: the edge suite's first case passed its
  assertion and the run still exited 23 on the leak checker.  Filed as
  `memory-perceus/a-method-called-on-a-qualified-enum-constructor-strands-its-string`;
  put back when it closes.

The new tests are in `tests/chrono_edge_tests.nv` — the sentence every
`ChronoError` prints, the instants a `CivilDate` cannot hold, a wait
with no monotonic reading to land on, and the host zones an ordinary
host never has — because the API suites are the published oracle and
are not edited.  It sets `TZ` for the cases that need one, so its
verdicts do not depend on the machine it runs on.

### Changed

- **Eight `== false` comparisons in `tests/mono_tests.nv` are spelled
  `not`.**  The 0.20.2 linter raises `redundant-bool` on each and
  suggests exactly that rewrite, which is the same predicate.  All
  eight are the interface release's: the 0.0.5 copy of the file lints
  to the same eight, at the same sites.  The assertions' meanings are
  untouched, and the suite still runs 10 of 10.

### Known

- Two lines in `src/zone.nv` are excused with a `cov: skip` marker:
  the `-1` arm of the private `host_seconds_of`, and the `NoLocalZone`
  arm in `offset_at` that consumes it.  Both rest on one measured host
  fact: glibc's `mktime` places every year the calendar can name
  rather than answering `-1`, so the refusal waits for a narrower
  `time_t`.  Merged coverage of `src/` is 128 of 130 lines (98.5%),
  and `novo pkg publish` accepts the two markers.
- A `Tick` starts near zero in a new process rather than at any
  machine-wide origin, so two readings are comparable only within one
  process.  That is what the type promises and what `std.time.Mono`
  already does, and the suite asserts nothing across processes.

## 0.0.5 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- calendar-nv: `^0.0.2` to `^0.2.0`.

## 0.0.4 — 2026-09-15

- README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.3 — 2026-09-10

- **Toolchain floor is 0.8.9**: the bodies and signatures use what 0.8.9 added (`todo()`, a bound effect parameter, the four layers), and the manifest says so instead of letting an older toolchain fail on an undefined function.  No signature changed.

## 0.0.2 — 2026-09-09

- **Dependencies are registry ranges**, not paths: the interface release 0.0.1 shipped a manifest whose dependencies pointed at sibling directories that exist only in the monorepo, so a consumer resolved the closure and then could not load the dependency.  No signature changed.

## [0.0.1] — 2026-09-09

**The interface, published before anyone implements it.** Every public
type and function carries its full signature, its effect row and its
doc comment; every body is `todo()`; the release is recorded
`implemented = false`.

### Added

- `zone` — `Offset`, minutes east of UTC, and three ways to get one:
  `utc()` with no row, `offset_at(when)` at `[io]` for the host's zone
  at a caller-supplied instant, and `local()` at `[time, io]` for the
  offset right now. The split is the package's load-bearing decision:
  a caller who already has an instant is not charged for a clock.
- `clock` — the wall clock, with no bare `now()`. `now_utc`, `now(at)`
  and `now_local` say which wall they are reading; `today` has the same
  three; `unix_seconds` and `unix_nanos` give the instant with no
  calendar on it; `from_unix` and `to_unix` are pure conversions that
  live here because they need an `Offset`.
- `mono` — the clock that only goes forward. `Tick`, `read`, `elapsed`,
  a pure `between`, and a blocking `sleep`. No conversion from a `Tick`
  to a date, ever.
- `timer` — `Deadline` over the monotonic clock and nothing else, with
  a pure `deadline_at` for a caller who has a reading; `Timer` as a
  stopwatch that carries its label so a report cannot pair the wrong
  two.
- `bridge` — conversions to and from `std.time`'s `Zoned`, `Duration`
  and `Mono`, with the two lossy directions written down rather than
  discovered.
- `chronoerror` — five reasons, one of which carries `calendar-nv`'s
  own error unchanged, so a caller can tell an impossible value from an
  environment that could not answer.

### Known

- The monotonic reading is `Tick` rather than `Instant`: `Instant` is a
  standard-library struct name and redeclaring one does not compile.
- Depends on `calendar-nv ^0.0.1`, which is itself an interface
  release. Both go to `0.1.0` together.
- `novo test` is red, and that is the release's expected state: every
  assertion in the API suite reaches `not implemented: chrono-nv.<fn>`.
  Run it with `--isolate` for one verdict per test naming the function
  it stopped at.
