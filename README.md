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
$ twill test tests
ok    tests\baseline_test.tw
ok    tests\harness_protocol_test.tw
ok    tests\report_test.tw
ok    tests\stats_test.tw
ok    tests\suite_test.tw

5 file(s): 5 passed, 0 failed
```

The example runs too, end to end, and exits 0:

```bash
twill run examples/tensor_ops.tw
```

`spool.toml` pins `^1.7.0` and CI installs v1.7.1, which is what the output in
this file was produced by. `docs/needs.md` is still worth reading -- it is the
list of what this library asked the language for, and it records which of those
arrived and which are still open.

## Getting twill

There is no build step: bobbin is twill source, so the only thing to install is
the compiler. Download a release binary, replacing `<os>-<arch>` with one of
`linux-amd64`, `linux-arm64`, `darwin-amd64`, `darwin-arm64`, or
`windows-amd64.exe`:

```bash
curl -fsSL -o twill \
  https://github.com/twill-lang/twill/releases/download/v1.7.1/twill-v1.7.1-<os>-<arch>
chmod +x twill
./twill --version
```

which prints:

```
Twill 1.7.1
```

Then, from the root of a clone of this repository:

```bash
twill test tests
twill run examples/tensor_ops.tw
```

Import paths resolve relative to the working directory, so both commands have to
be run from the project root. That is twill's rule rather than bobbin's.

## What bobbin is

| Piece | State |
| --- | --- |
| Timing harness: warmup, batched samples, adaptive stopping | runs. `tests/harness_protocol_test.tw`, and `examples/tensor_ops.tw` drives it for real |
| Median and interquartile range, never mean and sigma | runs. `tests/stats_test.tw` |
| Outliers counted and reported, never discarded | runs. `tests/report_test.tw`, and the example prints the counts |
| Clock overhead and granularity measured, not assumed | runs, and the measurement is the problem: see below |
| Regression tracking against a stored baseline, two-condition threshold | runs against an in-memory baseline. `tests/baseline_test.tw`. No baseline is loaded from disk yet |
| Human table and line-delimited JSON, as separate documents | runs. `tests/report_test.tw`, and the example prints both |
| Memory and allocation counters | wired, and off by default. `mem_counters_available()` is true on twill 1.7.1; set `opts.measure_memory = true` and `src/report.tw` prints allocs and bytes per iteration. `mem_tensors()` still returns -1, so the tensor count is reported as not counted |
| Reading and writing the baseline file | unblocked in twill 1.7, not wired into bobbin. `write_file` and `read_file` both exist and work; `report.render_baseline` still only returns the text, nothing writes it, and nothing parses it back |
| Flame graphs, sampling profiles, per-line attribution | **not in v0.1** |
| Anything running end to end | yes. `twill run examples/tensor_ops.tw` exits 0 |

The clock row is the one to read. `clk.probe` is called with 64 samples inside
`src/harness.tw`, and on the Windows machine this file was checked on 64 samples
are not enough to see a tick: `probe` reports a granularity of -1, meaning not
measurable, and every result carries the warning that says so. Called with
100,000 samples the same probe on the same machine reports an overhead of 254ns
and a granularity of 378,100ns. A 378us tick is why most of the example's
medians below are zero. See `docs/needs.md` entry 1.

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

`twill run examples/tensor_ops.tw`, on twill 1.7.1, on Windows, verbatim:

```
tensor ops, 128x128
benchmark                         median         iqr         min     n
matmul                          522.65us    373.02us         0ns  1000
    warning: the clock's granularity could not be measured; treat this timing as an upper bound on resolution, not as a measurement
    88 of 1000 samples above the outlier fence; max was 45.86ms
    mean 836.12us (for total wall time, not for comparison)
matvec                               0ns         0ns         0ns  1000
    warning: every sample was below one tick of the clock; raise inner
    101 of 1000 samples above the outlier fence; max was 1.38ms
    mean 58.21us (for total wall time, not for comparison)
softmax                              0ns    516.75us         0ns  1000
    warning: every sample was below one tick of the clock; raise inner
    10 of 1000 samples above the outlier fence; max was 9.22ms
    mean 246.80us (for total wall time, not for comparison)
sort                                 0ns         0ns         0ns  1000
    warning: every sample was below one tick of the clock; raise inner
    25 of 1000 samples above the outlier fence; max was 40.51ms
    mean 54.61us (for total wall time, not for comparison)
grad                              6.02us      4.12us         0ns   501
    warning: the clock's granularity could not be measured; treat this timing as an upper bound on resolution, not as a measurement
    45 of 501 samples above the outlier fence; max was 110.76us
    mean 7.80us (for total wall time, not for comparison)
```

Four of the five medians are zero and every row carries a warning, which is the
table working. `mono_ns` on this machine ticks about every 378us, `auto_inner`
sizes a batch for a millisecond that the clock cannot see, and the harness says
so on every row rather than printing the zeros bare. A table of numbers from
this run is not a measurement of matvec. Do not publish one.

## Machine output

One JSON object per benchmark, one per line. Line-delimited rather than one
array, so a run that is killed half way through still produces a file every line
of which parses, and appending a run is appending lines. It is a separate
document from the human table, not the same table with the punctuation removed:
a format that is sometimes parseable is a format nothing parses.

The first line of the same run as the table above:

```json
{"name":"matmul","unit":"ns","median":522650,"q1":508200,"q3":881225,"iqr":373025,"min":0,"max":45862500,"mean":836124.9,"robust_sigma":276519.644181,"samples":1000,"outliers":88,"iters":1000,"inner":1,"warmup":3,"clock_granularity_ns":-1,"clock_overhead_ns":0,"memory_available":false,"allocs_per_iter":0,"bytes_per_iter":0,"tensors_counted":false,"tensors_per_iter":null,"warning":"the clock's granularity could not be measured; treat this timing as an upper bound on resolution, not as a measurement"}
```

`memory_available` is `false` there because `measure_memory` is off by default,
not because the runtime has nothing to say. twill 1.7.1 has the counters and
bobbin calls them; turn the option on and the same field reads `true`, with real
figures beside it:

```json
{"name":"eltwise","unit":"ns","median":0,"q1":0,"q3":8064,"iqr":8064,"min":0,"max":22792,"mean":3431.586433,"robust_sigma":5977.761305,"samples":457,"outliers":2,"iters":29248,"inner":64,"warmup":1,"clock_granularity_ns":-1,"clock_overhead_ns":0,"memory_available":true,"allocs_per_iter":18,"bytes_per_iter":3331,"tensors_counted":false,"tensors_per_iter":null,"warning":"every sample was below one tick of the clock; raise inner"}
```

Whichever way it reads, every memory field is omitted from the human table when
`memory_available` is false, and `tensors_per_iter` is `null` rather than `0`
while `mem_tensors()` returns its -1. A zero in a benchmark table is a
measurement, and "we could not measure" is not zero.

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

## Using bobbin from another package

```
spool add bobbin https://github.com/twill-lang/bobbin
```

spool vendors into `twill_modules/`, and twill's import is a path, so the import
lines are the long ones above and they resolve relative to the project root.
That is twill's rule rather than bobbin's; see spool's README.

To work on bobbin itself, clone it and see "Getting twill" above. There is
nothing else to install: no build, no third-party packages, no Go.

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
