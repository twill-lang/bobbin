# Changelog

## v0.1.0 (unreleased)

First cut of bobbin, the benchmarking and profiling tool for twill, written in
twill.

It does not run. twill's `mode systems` is still being built, and bobbin needs a
monotonic nanosecond clock and allocation counters on top of it. See
`docs/needs.md` for the full list and `README.md` for the status table. Nothing
below has ever executed.

Added:

- A timing harness with mandatory warmup, batched samples, and stopping on both
  a minimum sample count and a minimum elapsed time, with ceilings so a slow
  benchmark terminates.
- Median and interquartile range reported everywhere, with the minimum, the
  outlier count and a robust sigma of `IQR / 1.349`. The mean is reported last
  and labelled as answering a different question.
- Clock overhead and granularity measured rather than assumed, and a warning
  attached to any result whose median is within twice the granularity.
- Memory and allocation counters behind an `available` flag, so a result can
  say "not measured" rather than print a zero. Nothing satisfies the flag yet.
- Regression tracking against a stored baseline with a two-condition threshold:
  a relative floor of 5% and 3 robust sigmas, both required. A run too noisy to
  support a verdict reports inconclusive rather than pass, and inconclusive
  does not fail a gate.
- Improvements reported rather than passed over, and baseline entries with no
  matching result named, because a rename resets a benchmark's history.
- Two outputs: an aligned human table, and line-delimited JSON as a separate
  document rather than the same table with the punctuation removed.
- The fixed ML workload suite (`suites/ml.tw`): mlp forward and
  forward-plus-backward, conv2d forward and forward-plus-backward, an
  attention block forward, softmax plus cross-entropy, an Adam step, and a
  memory-bandwidth-bound elementwise chain, each at fixed shapes and seeds
  with a dtype axis of f64, f32 and bf16.
- The suite's methodology (`docs/ml-workloads.md`): the PyTorch and NumPy
  reference implementations as listings, the analytic traffic and working-set
  accounting, the warmup and repetition protocol, and the honesty rules that
  make a comparison publishable.

Known gaps, deliberate for v0.1:

- No colour. twill's terminal layer is not reachable from an installed package;
  `docs/needs.md` entry 7.
- No file reading or writing, so the baseline can be rendered and not stored;
  `docs/needs.md` entry 4.
- No flame graphs, no sampling profiler, no per-line attribution.
