# proptest-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`. Installing this package works;
calling it panics with `not implemented`.

## What this is

The runner. It takes a strategy from
[`proptest-core-nv`](https://github.com/novolang/proptest-core-nv) and a
property, runs N cases from a seed you give or one it draws, shrinks the
first failure to its minimal case, and reports it **as a value** with
the seed that reproduces it — through `std.test`, so a property failure
reads like every other test failure.

`proptest-core-nv` does the generating and shrinking and has no
randomness in it. This one has the one draw.

```
novo pkg add proptest-nv
novo pkg build
novo test
```

## The one example that will work

```novo
use runner
use strategy
use propcheck
use std.list

fn involution(xs: [Int]) -> PropCheck
    propcheck.holds_if(list.reverse(list.reverse(xs)) == xs, "not an involution")

@test
fn test_reverse_is_an_involution() [fs, rand]
    runner.check_test("reverse twice",
                      strategy.list_of(strategy.int_range(0 - 99, 99), 0, 32),
                      runner.defaults(),
                      involution,
                      xs => "${list.len(xs)} elements")
```

A failure comes out where every other test failure does:

```text
  ✗ test_reverse_is_an_involution
      reverse_tests.nv:14:8: case reverse twice: 3 elements — not an
      involution (seed 8134502219, choices 2,1,0)
```

and `seed 8134502219 choices 2,1,0` is the line you paste into
`runner.replay` to get the case back.

## The load-bearing interface

```novo ignore
pub fn check<T>(g, cfg, prop: fn(T) -> PropCheck)      -> PropVerdict<T> [fs, rand]
pub fn check_io<T>(g, cfg, prop: fn(T) -> PropCheck [io]) -> PropVerdict<T> [fs, rand, io]
pub fn check_seeded<T>(g, cfg, seed: Int, prop)        -> PropVerdict<T>
pub fn replay<T>(g, choices: [Int], prop)              -> PropVerdict<T>
```

Not a type — the **effect rows**, which are four different answers to
"what did this run have to touch".

- `[fs, rand]` is **one draw**: `rand-nv`'s `os_u64` reads the operating
  system's random device, which is a file on the platforms this runs on,
  so both labels are honest. Every case, every shrink candidate and
  every replay after that is a function of the integer it returned.
- `check_seeded` and `replay` declare **nothing**. Reproducing a
  failure needs no entropy, no clock and no file, so a person can
  investigate one inside a function that is charged nothing, and a
  `@test` that pins a known counterexample has an empty row.
- `check_io` is for a property that performs. Its row is the caller's
  to declare and this package passes it through.

**Why not one effect-polymorphic entry.** The shape
[the publishing guide](https://novo-lang.org/docs/publishing.html#design)
recommends for "charged what the caller costs" is a bound effect
parameter, `fn check<T, P: Property[e]>(…) [e]`. It cannot be written:
SPEC § 3.6's grammar is `trait_bound ::= TYPE_IDENT [ '[' ident ']' ]`,
so a bound carries an **effect** argument and never a **type** one, and
a property trait generic over the value type has no spelling at all. A
function-typed parameter with a concrete row — `fn(T) -> PropCheck [io]`
— does compile, so the surface is two named entries with two honest rows
rather than one with a variable in it.

## Three verdicts, not two

| | means | `held()` |
| --- | --- | --- |
| `PropPassed(cases, seed)` | every case held | true |
| `PropFailed(failure)` | a case broke it; this is the minimal one | false |
| `PropExhausted(ran, skipped, seed)` | too few usable cases to conclude anything | **false** |

The third is the diagnosis a person needs when a `filter` is too tight
or an `assumed` is never true. "Your generator does not produce the
values your property is about" is a different problem from "your
property is wrong", and a runner that reported it as a pass would hide
it completely.

**Every verdict carries the seed, a pass included.** A run that passed
today and fails tomorrow is worth re-running at yesterday's seed, and a
seed nobody recorded is a seed nobody has.

## Why `show` is required

`check_test` takes `show: fn(T) -> Str`, and that is not an oversight.
`T` is generic; there is no `Debug`-for-everything to bound it by that a
caller's own type would satisfy; and a minimal counterexample nobody can
read is a counterexample nobody can act on. `_ => ""` is allowed and
leaves the seed and the transcript, which still reproduce it.

## The corpus is off by default

proptest writes a `.proptest-regressions` file; hypothesis keeps a
`.hypothesis` database. Both do it by default and both surprise people:
a file appears in a repository nobody added, a build agent's cache
decides which cases run, and a test that "passes on my machine" passes
because of a file that is not in the checkout.

So `PropConfig.case_file` is `None`, and a program that wants the
corpus names the path itself — which puts the path in that program's own
source, where a reader can see it. The format is one case per line in
`propcheck.replay_line`'s spelling: **one format, three uses** — the log
line, the file, and the argument to `replay`. A line that does not parse
is skipped with the file otherwise intact, because a corpus is a cache
and a corrupt cache should not stop a test run.

## The layer, and why

`host` — the one draw lives here, which is the point of the split.
`proptest-core-nv` is `core` and has no randomness at all; this package
depends on it and on `rand-nv`, which is the direction the layer order
allows.

`rand-nv` rather than `std.rand`, for one reason: `rand.seed(n)` sets a
**process-wide** seed, so two property tests in one process would
interfere and a third caller's `rand.shuffle` would move both. rand-nv's
operating-system half hands back an integer and keeps no state, which is
what a runner wants — draw once, and own the determinism from there.

There is no `[time]` anywhere, and the fallback seed when a platform
will not answer is a constant rather than a clock reading: a run that
could not read entropy should still be deterministic, and a time-based
seed would replace one source of non-determinism with another.

## The names, and the ones that were taken

| here | the obvious name | why not |
| --- | --- | --- |
| `PropConfig` | `Config` | `Config` is a struct in the orbit tree already |
| `PropVerdict<T>` | `Outcome`, `Report`, `Result` | the first two are taken and the third is the language's |
| `SavedCase`, `CaseFileFault` | `Case`, `Error` | `Error` is a standard-library trait, and `Case` is what `test.case` calls a label |
| `PropPassed`, `CaseWriteFailed` | `Passed`, `WriteFailed` | enum **variants** collide by bare name across the whole assembly |
| `seedsrc`, `casefile` (modules) | `rand`, `seed`, `file` | `rand.nv` is `std.rand`'s and `rng.nv` is rand-nv's |

## The reference implementation

**proptest** for the runner's shape: a case count, a shrink budget, a
seed in the failure report, and a persisted regression file. **hypothesis**
for the exhausted verdict and for counting discards separately from
failures. Rust's `#[test]` integration for the insistence that a property
failure is reported by the ordinary test reporter rather than by a
parallel one.

Deliberately left out, and where it went instead:

- **Generating and shrinking.** `proptest-core-nv`.
- **Parallel case execution.** A property is usually cheap and a
  parallel runner makes a failure's ordering non-deterministic, which
  is the one thing this package is built not to be.
- **A `--proptest-cases` command-line flag.** A library does not read
  the command line for its embedder; a program that wants one passes a
  `PropConfig`.
- **Timing out a case.** It needs `[time]` and a way to interrupt, and
  a property that hangs is a bug the hang itself reports.
- **Coverage-guided or corpus-minimising fuzz loops.** A different tool
  with a different budget.

## Status

Every function is `todo()`. Two suites, and they are red for two
different reasons.

**`tests/casefile_tests.nv`** reaches `not implemented:
proptest-nv.<fn>` on every assertion — the expected result until the
bodies land. It covers the corpus format, the configuration and the
seed source, which are the parts of this package that do not go through
a strategy.

**`tests/runner_tests.nv` does not compile, and `novo doc` reports the
examples that build a strategy as failing**, and that is one finding
rather than an oversight. Every runner entry takes a `PropStrategy<T>`,
and a generic function whose signature applies a generic head to its
type parameter is not emitted for a call from another module — so
`strategy.list_of(…)` cannot be called from here, or from any consumer.
It is filed against the LLVM backend as
`a-generic-fn-returning-a-generic-struct-is-not-emitted-for-a-cross-module-call`,
and proptest-core-nv's README has the four faces of it. The examples
are left as `novo` blocks rather than fenced `novo ignore`, because
they are code that should compile and fencing them would hide the
finding. When the defect closes, both suites and every example go green
with no edit.

| module | functions | rows | implemented |
| --- | --- | --- | --- |
| `verdict` | 5 | none | no |
| `runner` | 11 | none, `[fs, rand]`, `[fs, rand, io]` | no |
| `seedsrc` | 4 | none, `[fs, rand]` | no |
| `casefile` | 6 | none, `[fs]` | no |
