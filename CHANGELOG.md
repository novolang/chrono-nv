# Changelog

All notable changes to chrono-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

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
