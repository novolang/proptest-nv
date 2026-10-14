# Changelog

All notable changes to proptest-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.0 — 2026-10-13

The **implementation**.  Every `pub fn` body in `src/` is real, the two
shipped suites are the oracle, and `stability` moves from `draft` to `experimental`.  The interface is
unchanged from 0.0.4: same names, same signatures, same effect rows.

### Added

- `runner.check` runs the strategy for `cases` draws, judges each case,
  and returns a verdict; a case that breaks the property is shrunk
  through `proptest-core-nv`'s passes before the verdict is built, so
  `PropFailed` carries the smallest failing case the search reached.
- `runner.check_io` for a property that performs `[io]`, `check_seeded`
  for a run fixed at a seed the caller supplies, and `replay` for one
  recorded choice sequence.  `check_seeded` and `replay` still declare
  no effects.
- `runner.check_test` and `runner.report_test` report a verdict through
  `std.test`, so a property failure is an ordinary test failure with a
  message, a seed and a choice sequence.
- `casefile.record` appends a failing case to a corpus, making its
  parent directory, and answers the number of cases now in the file; a
  choice sequence the file already holds is not written a second time.
- `tests/proptest_nv_edge_tests.nv` — the edge suite this release adds
  beside the two shipped ones. It pins what this package is its own
  oracle for: a run performs exactly the number of cases it was asked
  for and no more, shrinking terminates and strictly decreases under
  `shrink.simpler_than`, a failure is reproducible from its seed alone
  and from its transcript with no seed at all, `max_shrinks` counts
  candidates OFFERED rather than accepted (five offers report the case
  as drawn and six report a shrink, on both the pure and the performing
  walk), the failure's case count names the case that broke, the corpus
  deduplicates on the seed and the transcript together rather than on
  either one, and a corpus fault names the path it was about.

### Changed

- `novo` floor `>= 0.8.9` to `>= 0.20.2`: the version floor is now the
  toolchain this release was verified against, because the bodies use
  constructs the interface release never compiled.
- The interface-release note is gone from `novo.toml`, and the stale
  "declared by path while the two are developed together" comment on
  the `proptest-core-nv` dependency is deleted.  The dependency is and
  was `^0.1.0`, resolved from the registry.
- `README.md` loses its status block and its "implementation status"
  table, and its failure message mock now matches what the runner
  actually prints: the seed and the choice sequence sit inside one
  parenthesised group, `(seed 8134502219 choices 2,1,0)`.

### Known

- **A match whose arms are a call does not get the expected type on
  this toolchain.** `verdict_of`, `verdict_pure` and `verdict_io` bind
  each arm's result to an annotated `let` rather than returning the
  call directly; without the binding the compiler reports E2020
  ("arms disagree … produces `V<Int>` … this one produces `V<_>`")
  inside one module and E2016 ("cannot infer `T`") across a module
  boundary.  Filing:
  `expected-type-closure-stops-at-the-arms-of-a-match-or-an-if`.
- **A match on the result of a call through a function-typed value
  leaks one box on 0.20.2.** `outcome_of`, `outcome_of_io` and
  `replay` bind the property's answer to a `let` first and match the
  binding.  Matching the call in place made `novo test` print every
  verdict green and still exit 23 with
  `the run leaked (exit 23 above)`.  The shape is recorded in
  `a-match-that-rebuilds-the-result-of-a-call-through-a-fn-typed-struct-field-leaks`,
  whose fix landed at `v0.20.2-23-g64da89386` and is therefore in
  0.20.3, not in the installed 0.20.2; fold the bindings back when this
  package's `novo` floor reaches it.
- **A struct literal is not written module-qualified on 0.20.2.**
  `src/runner.nv` imports `PropBroke` and friends from `propcheck` by
  name rather than spelling them `propcheck.PropBroke`.  Filing:
  `a-qualified-struct-literal-is-a-parse-error`, `fixed_in: 0.20.3`.
- The 0.0.1 blocker is closed: the generic function whose signature
  applies a generic head to its type parameter —
  `a-generic-fn-returning-a-generic-struct-is-not-emitted-for-a-cross-module-call`,
  a duplicate of `xmodule-generic-fn-generic-struct-never-monomorphized`
  — was fixed upstream on 2026-09-11.  `tests/runner_tests.nv` now
  compiles and all twelve of its cases pass.
- The budget boundary the edge suite pins — five offers report the case
  as drawn at those seeds and six report a shrink — is a measurement of
  `proptest-core-nv` 0.1.5's pass order and not a law of the runner: the
  passes are tried longest-deletion first, and `list_of` refuses the
  candidates that delete its length, so the first five offers at those
  seeds are discarded. A dependency release that reorders the passes
  moves the constant; the rule under test — `max_shrinks` counts
  candidates OFFERED, `>=` and not `>`, and not candidates accepted —
  is what must stay true, so re-measure the constant rather than the
  rule.
- The effect-polymorphic entry the publishing guide recommends —
  `fn check<T, P: Property[e]>(…) [e]` — still has no spelling, because
  a trait bound carries an effect argument and never a type argument
  (SPEC section 3.6).  Two named entries with concrete rows instead,
  as the interface release already decided.

## 0.0.4 — 2026-09-28

The dependency ranges move to the dependencies' current releases.  A
pre-1.0 caret range admits only the release it names, so the old
ranges held this package on interface releases, and a program could
not take this package beside those packages' current releases.  No
signature in this package changed.

- proptest-core-nv: `^0.0.1` to `^0.1.0`.

## 0.0.3 — 2026-09-25

The example property now reverses with `list.rev`, in the tests, in the
documentation of `check` and `check_test`, and in the README.  Under
novo 0.10.0 `list.reverse` reverses the list it is given in place.  A
property that reversed a list twice that way compared the list with
itself, so it could not fail.  `list.rev` answers a new list.  No change
to the interface.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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
