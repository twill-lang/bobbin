<p align="center">
  <img alt="twill" src="https://raw.githubusercontent.com/twill-lang/twill/main/assets/twill-mark.png" width="120">
</p>

<h1 align="center">bobbin</h1>

<p align="center">
  <b>Benchmarking and profiling for <a href="https://github.com/twill-lang/twill">twill</a>.</b><br>
  Written in twill.
</p>

<p align="center">
  <img alt="bobbin" src="https://img.shields.io/badge/bobbin-v0.1.0-7FE3C4?style=flat-square&labelColor=12332C">
  <img alt="written in twill" src="https://img.shields.io/badge/written%20in-twill-D2F0E4?style=flat-square&labelColor=12332C">
  <img alt="status" src="https://img.shields.io/badge/tests-passing-4FB79B?style=flat-square&labelColor=12332C">
  <img alt="MIT" src="https://img.shields.io/badge/license-MIT-D2F0E4?style=flat-square&labelColor=12332C">
</p>

---

## It runs

`bobbin` is written in twill, in `.tw` files, using `mode systems`. That subset
did not exist when this library was written, so for a long time none of the code
here executed and this section said so. twill 1.6 is the release that closed it:
the 5 test suites under `tests/` pass, and CI runs them against a released
twill on every push rather than gating on the prose in this file.

```bash
twill test tests
```

You need twill 1.7.0 or newer. `docs/needs.md` is still worth reading -- it
is the list of what this library asked the language for, and it now records
which of those arrived and which are still open.

## What bobbin is

| Piece | State |
| --- | --- |
| Timing harness: warmup, batched samples, adaptive stopping | written, unrun |
| Median and interquartile range, never mean and sigma | written, unrun |
| Outliers counted and reported, never discarded | written, unrun |
| Clock overhead and granularity measured, not assumed | written, unrun |
| Regression tracking against a stored baseline, two-condition threshold | written, unrun |
| Human table and line-delimited JSON, as separate documents | written, unrun |
| Memory and allocation counters | **blocked.** The runtime exposes none |
| Reading and writing the baseline file | **blocked.** No file writing |
| Flame graphs, sampling profiles, per-line attribution | **not in v0.1** |
| Anything running end to end | **no** |

## Median and IQR, not mean and sigma

A timing distribution has a hard floor and no ceiling. There is a fastest the
work can possibly be done, and every source of interference (a preemption, a
page fault, a cache eviction by another process, a frequency step) can only make
a sample slower. The distribution is right-skewed with a long tail, on every
machine, always.

On that shape the mean sits above the mode and moves with the tail rather than
with the work. One preemption in a hundred runs adds a sample twenty times the
median and shifts the mean by twenty percent, and the tool reports a twenty
percent regression in code nobody changed. The standard deviation is worse,
because it squares the deviations: the interval that is supposed to tell you
whether the change is real is widened by the same outlier that caused it. Mean
and sigma describe a symmetric distribution, and this one is not.

So bobbin reports:

- **median**, whose position does not move when a tail sample gets worse;
- **interquartile range**, the spread of the middle half, where the work is;
- **minimum**, the closest observable thing to the interference-free cost, so a
  large gap between minimum and median says the machine got measured;
- **outlier count**, because outliers are counted and reported, never removed.
  Discarding the slow samples is how a benchmark stops being able to see the
  interference it exists to warn about;
- **mean**, last and labelled, because total wall time for n runs is n times the
  mean and never n times the median. Different question, printed as one.

`stats.robust_sigma` is `IQR / 1.349`, which equals the standard deviation for a
normal distribution and does not blow up when one sample is twenty times the
rest. That is what makes it usable in a threshold written in sigmas.

## The measurement protocol

```rust
mode systems

import "twill_modules/bobbin/src/suite.tw" as suite
import "twill_modules/bobbin/src/harness.tw" as bench
import "twill_modules/bobbin/src/baseline.tw" as base

let A = randn(128, 128)
let B = randn(128, 128)

# Bodies return a value. The harness feeds it to `keep`, so the work is not
# removable the day twill has an optimiser that could remove it.
fn bench_matmul(i: I64) -> F64 = item(sum(A @ B))
fn bench_sort(i: I64) -> F64 = item(sum(sort(A)))

let s = suite.suite("tensor ops")
suite.add(s, "matmul", bench_matmul)

# For an operation fast enough that the clock is a real fraction of it, batch
# until a sample lasts about a millisecond.
let opts = bench.defaults()
opts.inner = bench.auto_inner(bench_sort)
suite.add_with(s, "sort", opts, bench_sort)

let results = suite.run_all(s)
suite.print_human(s, results)

let stored = base.new_baseline()
let cs = suite.compare_all(results, stored, base.default_thresholds())
suite.print_comparisons(cs, stored, results)
if suite.gate(cs) { exit(1) }
```

1. **Warmup runs are executed and discarded.** The first run of anything pays
   for a cold instruction cache, cold data, and lazily initialised state.
   Including it makes the first benchmark in a file look slower than the same
   benchmark run second, which is an ordering effect people lose afternoons to.
   Warmup is not zero by default.
2. **Each sample times a batch of `inner` iterations and divides.** When one
   operation costs less than a few hundred ticks, the timing call dominates it
   and what gets measured is the clock. `auto_inner` picks a batch size that
   lasts about a millisecond.
3. **Samples accumulate until both a minimum count and a minimum elapsed time
   are met.** A fixed count is wrong in both directions: 100 samples of a
   10-second operation is a 17-minute benchmark, and 100 samples of a 100ns
   operation is 10 microseconds of evidence.
4. **The clock is measured, not assumed.** `clk.probe` reports the overhead of a
   timing call and the smallest non-zero interval the clock can resolve. A
   median within twice the granularity is flagged, because a benchmark below the
   clock's floor reports a number that is about the clock.

## Human output

```
tensor ops, 128x128
benchmark                         median         iqr         min     n
matmul                            4.21ms      0.09ms      4.13ms    88
    mean 4.28ms (for total wall time, not for comparison)
softmax                          182.40us     4.10us    178.02us   400
    3 of 400 samples above the outlier fence; max was 2.94ms
    mean 195.71us (for total wall time, not for comparison)
sort                               1.94us     0.42us      1.51us  1000
    warning: interquartile range is 21.6% of the median; this machine is
    too noisy for a small threshold
    mean 2.08us (for total wall time, not for comparison)
```

## Machine output

One JSON object per benchmark, one per line. Line-delimited rather than one
array, so a run that is killed half way through still produces a file every line
of which parses, and appending a run is appending lines. It is a separate
document from the human table, not the same table with the punctuation removed:
a format that is sometimes parseable is a format nothing parses.

```json
{"name":"matmul","unit":"ns","median":4210344,"q1":4174000,"q3":4264000,"iqr":90000,"min":4131002,"max":5012773,"mean":4283910,"robust_sigma":66716,"samples":88,"outliers":2,"iters":88,"inner":1,"warmup":3,"clock_granularity_ns":41,"clock_overhead_ns":18,"memory_available":false,"allocs_per_iter":0,"bytes_per_iter":0,"tensors_per_iter":0,"warning":""}
```

`memory_available` is `false` today and every memory field is omitted from the
human table when it is. A zero in a benchmark table is a measurement, and "we
could not measure" is not zero.

## Regression tracking that does not cry wolf

A gate that fails on any change gets disabled within a week, and a disabled gate
catches nothing. A gate that is too strict is strictly worse than one that is
slightly loose.

A verdict of `regressed` needs **both**:

1. **A relative floor.** Default 5%. A one percent change is not worth attention
   regardless of how confidently it was measured, and on most machines it cannot
   be measured confidently anyway.
2. **Clearance of the measured noise.** Default 3 robust sigmas, using the
   larger of the two runs' `IQR / 1.349`, because the noisier of the two limits
   what can be concluded from either.

Condition 1 alone fails on a quiet machine measuring a real but irrelevant
change. Condition 2 alone fails on a very quiet machine, where a 0.5% shift is
many sigmas and still nothing.

Above a relative IQR of 10%, or when the harness attached a warning, the verdict
is `inconclusive` rather than `pass`. Reporting a pass from unusable data is what
lets a regression through, so the doubt resolves towards saying so. Inconclusive
does not fail a gate: a noisy runner is not a regression, and failing on one
teaches people to rerun until it passes, which defeats the gate for real
regressions too.

Improvements are reported, not passed over. An unexplained speedup is as likely
to be a benchmark that stopped doing the work as an optimisation, and this is
the cheapest moment to notice. A baseline entry with no matching result is named
too, because that is usually a rename, and a rename silently resets the history
of the thing renamed.

There is no significance test, deliberately. See `docs/needs.md` entry 13.

## Install

Once spool, `mode systems` and a clock all exist:

```
spool add bobbin https://github.com/twill-lang/bobbin
```

spool vendors into `twill_modules/`, and twill's import is a path, so the import
lines are the long ones above and they resolve relative to the project root.
That is twill's rule rather than bobbin's; see spool's README.

## Repository layout

```
src/clock.tw        the clock interface, and what the runtime has to provide
src/stats.tw        order statistics, and the argument for them
src/memory.tw       allocation counters, and what they need
src/harness.tw      warmup, batching, sampling, diagnosis
src/baseline.tw     stored entries, thresholds, verdicts
src/report.tw       the human table, the JSON lines, the baseline file
src/suite.tw        registering and running a set of benchmarks
tests/              tests, named as sentences, none of which needs a clock
examples/           a suite over five tensor operations
suites/ml.tw        the fixed ML workload suite: eight workloads, fixed
                    shapes and seeds, a dtype axis of f64, f32 and bf16
docs/ml-workloads.md  the ML suite's methodology: the PyTorch and NumPy
                    references, what is measured, and the honesty rules
docs/needs.md       what the language and the runtime still have to provide
```

The tests deliberately never take a real timing. A benchmarking tool whose own
tests depend on what the machine was doing is a tool nobody can debug.

## Dependencies

twill, and nothing else. No third-party twill packages and no Go.

## Contributing

The most useful contribution right now is not code. It is a correction to
[`docs/needs.md`](docs/needs.md): a feature listed there that the language
already has, a workaround that is worse than described, or a missing entry found
by reading the source. After that, the threshold argument above is the part most
worth arguing with.

## License

MIT. See [LICENSE](LICENSE).
