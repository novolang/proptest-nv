# Changelog

All notable changes to proptest-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.1 — 2026-09-10

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `runner` — `PropConfig`, and four ways to run a property whose
  **effect rows are the disclosure**: `check` at `[fs, rand]` for the
  one seed draw, `check_io` at `[fs, rand, io]` for a property that
  performs, and `check_seeded` and `replay` at **no effects at all**,
  because reproducing a recorded case needs neither entropy nor a clock
  nor a file. `check_test` reports through `std.test`, so a property
  failure reads like every other test failure.
- `verdict` — three verdicts rather than two: a run that could not
  generate enough usable cases is **exhausted**, which is not a pass,
  and that is the diagnosis a person needs when a filter is too tight.
  Every verdict carries the seed, a pass included.
- `seedsrc` — the one draw, through rand-nv's operating-system half
  rather than `std.rand`, whose `seed` is process-wide and would let
  two property tests interfere. No `[time]` anywhere: the fallback seed
  is a constant, because a run that could not read entropy should still
  be deterministic.
- `casefile` — the persisted failure corpus, **off by default**, in
  `propcheck.replay_line`'s format: one format for the log line, the
  file and the argument to `replay`.

### Known

- **`tests/runner_tests.nv` does not compile.** Every entry takes a
  `PropStrategy<T>` from proptest-core-nv, and a generic function whose
  signature applies a generic head to its type parameter is not emitted
  for a call from another module. The filing is
  `a-generic-fn-returning-a-generic-struct-is-not-emitted-for-a-cross-module-call`.
  `casefile_tests.nv` covers everything that does not go through a
  strategy and reaches `not implemented` as an interface suite should.
- The effect-polymorphic entry the publishing guide recommends —
  `fn check<T, P: Property[e]>(…) [e]` — has no spelling, because a
  trait bound carries an effect argument and never a type argument. Two
  named entries with concrete rows instead.
- `proptest-core-nv` is declared by path while the two halves are
  developed together. It converts to `^0.0.1` before publish.
