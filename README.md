# proptest-nv

A property-based test checks that a stated fact holds for many generated
inputs rather than for a few written ones. This package is the runner:
it takes a strategy and a property, runs a number of cases from a seed,
shrinks the first failing case to its smallest form, and reports it
through `std.test` so a property failure reads like every other test
failure. The strategies and the shrinking come from
[proptest-core-nv](https://novo-lang.org/packages/proptest-core-nv). The
runner's shape follows
[proptest](https://docs.rs/proptest/latest/proptest/), and the treatment
of skipped cases follows
[Hypothesis](https://hypothesis.readthedocs.io/en/latest/).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What it is

A **property** is a fact that should hold for every input, such as
"reversing a list twice gives the list back". A **strategy** is a rule
for producing one input. A run pairs the two: the strategy makes a case,
the property judges it, and the runner repeats that until it has enough
passes or one failure.

A property answers with one of three values. `PropHolds` means it held.
`PropBroke(why)` means it did not, and carries the author's own words.
`PropSkip` means the case did not meet the property's precondition and
proves nothing either way. All three come from proptest-core-nv.

A run answers with a **verdict**, and there are likewise three.

| Verdict | Means | `verdict.held` |
| --- | --- | --- |
| `PropPassed(cases, seed)` | every case held | true |
| `PropFailed(failure)` | a case broke the property; this is the smallest such case found | false |
| `PropExhausted(ran, skipped, seed)` | too few usable cases to conclude anything | false |

The third is the answer a person needs when a `filter` is too tight or a
precondition is never true. "The generator does not produce the values
this property is about" is a different problem from "this property is
wrong", and a runner that reported it as a pass would hide it.

**Shrinking** is the search for a smaller failing case. The runner asks
proptest-core-nv for a candidate sequence of choices, replays it through
the same strategy, and answers whether the property broke on it in the
same way. What comes out is the smallest case the search found, together
with the seed and the choice sequence that reproduce it.

Randomness is used once per run, to pick the seed. Everything after that
is arithmetic on that one integer, so a printed seed reproduces the whole
run on any machine.

| Setting | Default | What it bounds |
| --- | --- | --- |
| `cases` | 256 | how many cases run before a pass is declared |
| `max_shrinks` | 1024 | how many candidates the shrink search offers |
| `max_skips` | 2048 | how many skipped cases before the run is exhausted |
| `seed` | none | a drawn seed; set one to fix the run |
| `case_file` | none | the path of a file of recorded failures |

## Install

```
novo pkg add proptest-nv
```

proptest-core-nv comes with it, so `use strategy` and `use propcheck`
resolve without a second entry in your manifest.

## Example

```novo
use std.list
use std.test
use propcheck
use runner
use strategy

// The property, as an ordinary function. The text is used only on a
// failure, and it is the sentence the report prints.
fn involution(xs: [Int]) -> PropCheck
    propcheck.holds_if(list.rev(list.rev(xs)) == xs,
                       "not an involution")

@test
fn test_reverse_is_an_involution() [fs, rand]
    // "reverse twice" labels the case in the failure message.
    // The strategy makes a list of up to thirty-two integers.
    // `defaults()` is 256 cases from a seed drawn once.
    // The last argument renders a failing value for the report.
    runner.check_test("reverse twice",
                      strategy.list_of(strategy.int_range(0 - 99, 99), 0, 32),
                      runner.defaults(),
                      involution,
                      xs => "${list.len(xs)} elements")
```

A failure appears where every other test failure does.

```text
  ✗ test_reverse_is_an_involution
      reverse_tests.nv:14:8: case reverse twice: 3 elements — not an
      involution (seed 8134502219, choices 2,1,0)
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a
`not implemented: proptest-nv.<module>.<fn>` panic, or one from
proptest-core-nv. The tests are the specification the implementation will
have to satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `runner` | The configuration of a run and the calls that perform one: `check`, the variant for a property that performs input or output, the two that reproduce a recorded case, and the two that report through `std.test`. |
| `verdict` | The three answers a run may give, and the readers that take a verdict apart: its name, whether it passed, its seed, its failure and its rendered message. |
| `seedsrc` | Where a seed comes from: the one draw from the operating system, the fallback when a platform will not answer, and a seeded stream for a caller who wants one. |
| `casefile` | The file of recorded failures: one case per line, reading it, appending to it, and turning the same text into values with no file on disk. |

## How to choose an entry point

**`runner.check_test` is the one to write in a `@test`.** It runs the
property and reports through `std.test`, so a failure is an ordinary test
failure with a message, a seed and a choice sequence.

**`runner.check` answers a verdict instead of reporting one.** Use it
when the program wants to branch on the result, count failures across
several properties, or print its own summary.

**`runner.check_io` is `check` for a property that performs input or
output.** A property that logs, writes to a database or shells out is an
honest property, and this is the entry that accepts one.

**`runner.check_seeded` and `runner.replay` reproduce.** `check_seeded`
re-runs a whole run at a seed a report printed. `replay` plays one
recorded choice sequence back. Neither draws entropy, so both can be
called from a function that declares no effects at all.

**`runner.report_test` and `runner.message_for` report a verdict you
already hold.** The first sends it to `std.test`, the second returns the
message as text for a reporter of your own.

## The rules a user needs

1. **A property returns a value; it does not assert.** Build the answer
   with `propcheck.holds_if` for an ordinary boolean, `propcheck.assumed`
   for a precondition, and `propcheck.all_of` for several assertions in
   one property. A property that calls `test.assert` itself bypasses the
   shrinking and reports the first failing case rather than the smallest.
2. **An exhausted run is not a pass.** `verdict.held` answers false for
   `PropExhausted`, and `check_test` fails the test. A property whose
   generator never produced a usable case has not been checked.
3. **Every verdict carries a seed, a pass included.** A run that passed
   today and fails tomorrow is worth re-running at yesterday's seed, and
   a seed nobody recorded is a seed nobody has.
4. **`show` is required, and `_ => ""` is a valid answer.** `check_test`
   renders the failing value with the function you give it. There is no
   universal way to print a caller's own type, so the runner asks for
   one. An empty string still leaves the seed and the choice sequence,
   which reproduce the case.
5. **The declared effects say what a call may touch, not what one
   argument makes it touch.** `check` declares `[fs, rand]` because it
   may draw a seed, and it declares them even when the configuration
   already fixes one. `check_seeded` and `replay` declare nothing,
   because a seed or a transcript the caller supplied needs no entropy,
   no clock and no file.
6. **Randomness is drawn once per run.** `seedsrc.draw_seed` reads the
   operating system's entropy, which is a file on the platforms this runs
   on, so the call declares both `[fs]` and `[rand]`. Every case, every
   shrink candidate and every replay after it is a function of that one
   integer.
7. **A platform that will not answer gets a constant, not a clock.**
   `seedsrc.draw_or` takes the fallback seed. A run that could not read
   entropy should still be reproducible, and a time-based seed would
   replace one source of non-determinism with another.
8. **The file of recorded failures is off until you name a path.**
   `PropConfig.case_file` is absent in `defaults()`. Set it with
   `runner.with_case_file` and the path is in your own source, where a
   reader can see it.
9. **An absent case file is an empty corpus, not a fault.**
   `casefile.load_cases` answers an empty list for a file that does not
   exist. A line it cannot parse is skipped and counted by
   `casefile.bad_lines`, because a corpus is a cache and a damaged cache
   should not stop a test run.
10. **A recorded case is appended, never overwritten, and never twice.**
    `casefile.record` keeps the failures a property has had, and it does
    not write a choice sequence the file already holds.
11. **One text format serves three purposes.** The line a failure report
    prints, the line a case file holds, and the argument a person pastes
    into `runner.replay` are the same spelling,
    `propcheck.replay_line`'s.
12. **A property that needs effects beyond `[io]` is run through
    `replay` in a loop you write.** `check_io` accepts a property
    declaring `[io]`, which covers printing, pipes and subprocesses. A
    property needing `[net]` or `[fs]` has no entry here, because a
    function-typed parameter must name its effects concretely and a
    parameter that varies over them cannot be spelled (SPEC
    section 3.6).

## What is not included

- **Generating and shrinking values.** Those are
  [proptest-core-nv](https://novo-lang.org/packages/proptest-core-nv).
  This package draws the seed and drives the search.
- **One entry point that adapts to whatever the property costs.** A trait
  bound carries an effect argument and never a type argument
  (SPEC section 3.6), so a property trait generic over the value type
  cannot be written. The surface is `check` and `check_io`, each naming
  its effects, rather than one entry with a variable in it.
- **A generic helper of your own that forwards to these functions.** A
  function you declare, generic over the same type parameter, that calls
  `runner.check` or another generic function here, fails to build: the
  compiler emits a call to a symbol it never generated. Call these
  functions directly, or give your helper a concrete value type such as
  `PropStrategy<[Int]>`. The defect is recorded as
  `generic-forwarding-to-a-package-generic-skips-monomorphisation`.
- **Running cases in parallel.** A property is usually cheap, and a
  parallel runner makes the order in which failures are found vary
  between runs.
- **A command-line flag for the case count.** A library does not read the
  command line on its embedder's behalf. A program that wants one reads
  it and passes a `PropConfig`.
- **A time limit on a case.** It needs a clock and a way to interrupt a
  running function. A property that hangs is reported by the hang.
- **Coverage-guided fuzzing and corpus minimisation.** Those need the
  compiler to instrument the program under test.
- **Running on a microcontroller.** No such claim is made. Running
  hundreds of cases and shrinking a failure is a development activity.

## Related packages

- [proptest-core-nv](https://novo-lang.org/packages/proptest-core-nv) is
  the strategies, the recorded choice sequence and the shrink passes. It
  declares no effects and contains no randomness. Take it on its own to
  generate or shrink values with no runner in the program.
- [rand-nv](https://novo-lang.org/packages/rand-nv) is where the one seed
  comes from. Its operating-system call answers an integer and keeps no
  state, which is what a runner needs. `std.rand` in the standard library
  is not used here, because `rand.seed` sets a seed for the whole
  process, so two property tests in one process would interfere.
- [fake-nv](https://novo-lang.org/packages/fake-nv) is fixture data built
  on the same strategies, so a fixture and a property draw from one
  source.
- [matchers-nv](https://novo-lang.org/packages/matchers-nv) is assertions
  whose failures explain themselves. A property body can use one and pass
  its description to `propcheck.holds_if`.
- `std.test` in the standard library is the test harness a novo-lang
  `@test` reports to. This package calls `test.case` and `test.fail` in
  `check_test` and `report_test`, and nowhere else.

## Tests

```bash
novo test tests/runner_tests.nv      # 12 tests: the run, the verdicts, the effects
novo test tests/casefile_tests.nv    #  8 tests: the corpus format and the seed draw
```

The reference implementations are proptest, for the case count, the
shrink budget, the seed in the failure report and the persisted
regression file; Hypothesis, for counting discarded cases separately and
for the exhausted verdict; and Rust's own test attribute, for the rule
that a property failure is reported by the ordinary test reporter rather
than by a parallel one.

The suite asserts that a passing run reports the seed it ran from, that a
failure is shrunk and carries what reproduces it, that a fixed seed makes
a run reproducible from the start, that `check_seeded` and `replay`
declare no effects at all, that a generator which never produces a usable
case is exhausted rather than passed, that an overrun case is discarded
and not judged, that the rendered message and the replay line agree, that
the corpus is off by default, that the corpus format round-trips with no
file on disk, that a corrupt line is skipped rather than fatal, that an
absent file is an empty corpus, that recording appends and does not
duplicate, and that the seed is drawn exactly once.

The tests compile today and fail at run, each on a `not implemented`
panic from this package or from proptest-core-nv. That is the expected
state of an interface release. They turn green one at a time as bodies
land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `runner.defaults`, `.with_seed`, `.with_cases`, `.with_case_file` | no |
| `runner.check`, `.check_io`, `.check_seeded`, `.replay` | no |
| `runner.check_test`, `.report_test`, `.message_for` | no |
| `verdict.verdict_name`, `.held`, `.seed_of`, `.failure_of`, `.render` | no |
| `seedsrc.draw_seed`, `.draw_or`, `.seed_for`, `.stream_from` | no |
| `casefile.load_cases`, `.record`, `.bad_lines` | no |
| `casefile.render_cases`, `.parse_cases`, `.case_fault_path` | no |
| `CaseFileFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
