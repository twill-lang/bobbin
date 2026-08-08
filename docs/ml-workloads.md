# The ML workload suite, and how to compare it honestly

`suites/ml.tw` is a fixed suite of ML workloads written as bobbin benchmarks.
This document is its methodology: how the same workloads are defined in PyTorch
and NumPy, what is measured, how the measurements are taken, and the rules that
make a comparison publishable. The suite exists because the twill ecosystem
wants to be able to claim that twill is efficient for ML work, and a claim like
that is worthless unless the suite behind it would show it false if it were
false. Everything below is arranged so that it could.

Where this document and `suites/ml.tw` disagree, the suite governs and this
document has a bug.

The suite does not run yet; nothing in bobbin does. That is not a reason to
defer the methodology, it is the reason to write it now: the first number
anyone publishes will be produced by whatever rules exist on that day.

## The workloads

Eight workloads, each at one fixed set of shapes, each measured at three
dtypes. The names are the benchmark names in the suite; the dtype point is the
part after the slash, so `mlp_fwd/bf16` is one benchmark.

| workload | shapes | what dominates |
| --- | --- | --- |
| `mlp_fwd` | X [32, 784], W1 [784, 256], W2 [256, 10], relu between | matmul |
| `mlp_fwdbwd` | the same, gradients for W1, B1, W2, B2 of an MSE loss | matmul, both directions |
| `conv2d_fwd` | X [16, 64, 64], W [32, 16, 3, 3], valid padding, unit stride | convolution |
| `conv2d_fwdbwd` | the same, gradients for X and W of the summed output | convolution, both directions |
| `attention_fwd` | X [128, 64], Wq, Wk, Wv [64, 64], one head | matmul and softmax |
| `softmax_xent` | logits [64, 1000], one-hot targets [64, 1000] | exp and log |
| `adam_step` | the mlp parameter tree, 203,530 parameters, one update | elementwise, memory |
| `eltwise_chain` | one vector of 2^24 = 16,777,216 elements, eight ops and a sum | memory bandwidth |

The dtype axis is `f64`, `f32`, `bf16` on every workload. f16 is deliberately
absent: the twill dtype design (`docs/dtypes.md` in the twill repository) says
to prefer bf16, f16 is unusable for training without loss scaling, and a suite
point nobody should run in practice is a suite point that only exists to be
quoted.

Approximate work per iteration, computed from the shapes. These are stated so
that a measured number can be sanity-checked against them, not measured:

| workload | FLOP | minimum traffic at f64 | working set at f64 |
| --- | --- | --- | --- |
| `mlp_fwd` | 13.0 M | 3.9 MB | 1.9 MB |
| `mlp_fwdbwd` | 39 M | 12 MB | 4 MB |
| `conv2d_fwd` | 35.4 M | 1.6 MB | 1.5 MB |
| `conv2d_fwdbwd` | 106 M | 5 MB | 3 MB |
| `attention_fwd` | 7.4 M | 0.9 MB | 0.5 MB |
| `softmax_xent` | 0.6 M | 3.6 MB | 1.6 MB |
| `adam_step` | 2.0 M | 14.7 MB | 11.4 MB |
| `eltwise_chain` | 151 M | 2.55 GB | 268 MB |

Traffic is the analytic minimum: every operation in the expression tree reads
each tensor operand once and writes its result once, at the element width of
the dtype. Nothing today can measure the real traffic from inside twill
(`docs/needs.md` entry 2), so the honest statement is "at least this many
bytes moved", and that is the statement the suite makes. Working set is the
peak of simultaneously live arrays.

For the chain the accounting is worth writing out, because the chain is the
workload whose entire point is the traffic. The expression is

    sum(sqrt(abs(relu(x * 1.25 + 0.25) * x - 0.125)) + x)

which is nine operations: five with one tensor operand, two with two (the
multiply by x and the add of x), one with one (abs), and the final sum. Reads
are 11 N elements, writes are 8 N, so 19 N elements move in total:

| dtype | bytes per element | minimum traffic per iteration |
| --- | --- | --- |
| f64 | 8 | 2.55 GB |
| f32 | 4 | 1.27 GB |
| bf16 | 2 | 0.64 GB |

Dividing the median iteration time into that traffic gives an effective GB/s,
and that number against the machine's measured memory bandwidth is the honest
utilisation figure. It is a lower bound on real traffic, so the utilisation it
yields is a lower bound too, which is the safe direction to be wrong in.

## Fixed seeds, and what they do and do not promise

Every workload draws its inputs under its own fixed seed (101 through 106, in
the suite), at f64, once, outside every timed body. The f32 and bf16 input
sets are the f64 draws cast once, the single rounding twill's dtype design
specifies for `.to`.

The seeds make a run reproducible within one implementation. They do not make
the values equal across implementations, because twill, PyTorch and NumPy do
not share a random number generator, and the suite does not pretend otherwise.
That is acceptable for timing on purpose: every kernel here is branch-free
over its data (relu and abs are elementwise selections, not branches), so its
running time does not depend on the values. Checking that twill computes the
same *numbers* as a reference is the differential harness's job in the twill
repository, against saved inputs, and is a different tool with different rules.

## The dtype axis, and what each point measures

The axis exists to measure exactly what the dtype work buys, which means being
precise about what it can show today:

- **f64** is twill as it exists: every tensor is f64 and always has been. This
  point is the baseline for the transition and the anchor for history.
- **f32** and **bf16** are the dtype semantics of `docs/dtypes.md`: values
  rounded on store, f32 accumulation for everything narrower than f32,
  gradients never narrower than f32.

Until twill's packed buffer lands (NEEDS-111 in the twill repository), a bf16
element still occupies 64 bits of memory. All three dtype points therefore
move identical bytes today, and the narrow points do strictly more work,
because they round. **Before NEEDS-111, the f32 and bf16 columns measure the
cost of dtype semantics, and a reader should expect them to be slower than
f64. That is the correct result, not a failed one.** The first run where the
dtype axis is a bandwidth axis is the first run after the packed layout lands,
and the before-and-after pair of baselines across that landing is the
measurement of what the layout bought. `docs/needs.md` entry 15 records this
so nobody misreads the first run.

## What is measured

Three things, and each is labelled with how it was obtained:

1. **Wall time.** Per-iteration nanoseconds, reported as median,
   interquartile range, minimum, sample count and outlier count, exactly as
   `src/stats.tw` argues and everything in bobbin reports. Never a mean alone,
   never a single number.
2. **Bytes moved.** The analytic minimum from the table above, stated as a
   minimum. If twill grows memory counters, measured bytes are reported next
   to the analytic figure, labelled, and the analytic figure stays, because it
   is the one that is comparable across runtimes.
3. **Working-set size.** Analytic, from the shapes and the dtype width. It
   says which cache level a workload runs from, which is the context a
   bandwidth number is meaningless without.

## Warmup and repetition

The suite uses bobbin's protocol unmodified, and the reference implementations
must implement the same protocol, because a comparison between two harnesses
is a comparison of the harnesses:

- **Warmup:** 3 batches, executed and discarded.
- **Batching:** a sample times `inner` iterations and divides. For the small
  forward passes and the optimiser step, `inner` is chosen by `auto_inner` so
  a sample lasts about a millisecond; for the fwdbwd pairs and the chain,
  `inner` is 1.
- **Stopping:** samples accumulate until at least 30 samples and at least one
  second of measured time, whichever is later, with ceilings of 1000 samples
  and 30 seconds.
- **Nothing discarded:** outliers are counted and reported, never removed.
- **Verdicts:** a change is a regression only if it is at least 5% and at
  least 3 robust sigmas (IQR / 1.349, the larger of the two runs). A run whose
  relative IQR exceeds 10% is inconclusive, and inconclusive is reported as
  inconclusive, not as pass and not as failure.

## The reference implementations

The references are listings in this document, not files in this repository.
bobbin ships no Python, and a runnable reference that drifted from the suite
would be worse than none. When the suite first runs, these listings are to be
transcribed as-is into whatever harness repository hosts the comparison, and
diffs from the listings are bugs in the transcription.

Rules the references follow, stated before the code so the code can be checked
against them:

- **Eager PyTorch, CPU.** No `torch.compile`, no GPU. A compiled or fused
  variant may be reported *in addition*, clearly labelled, never substituted:
  the measured cell is the same eager op-by-op execution twill performs.
  Record `torch.__version__` and `torch.get_num_threads()` with every run.
- **Explicit formulas, not fused library calls.** The cross-entropy cell uses
  the logsumexp formula below, not `F.cross_entropy`; the Adam cell uses the
  update written out, not `torch.optim.Adam` (which is fused and foreach by
  default). The fused forms may be reported additionally, labelled, for the
  same reason as above: they are interesting, and they are not the same math
  under the same discipline.
- **NumPy is f64 and f32 only.** NumPy has no bf16. The bf16 cells of the
  NumPy column are reported as absent, not simulated with `ml_dtypes`, whose
  accumulation behaviour is not the one twill specifies. An absent cell is a
  fact; a simulated one is an argument.
- **NumPy has no autodiff.** The mlp backward is written out by hand below,
  because it is six standard lines. The conv2d backward is not: a hand-rolled
  im2col backward would benchmark the transcription, not NumPy, so
  `conv2d_fwdbwd` is absent from the NumPy column and says so.

### The harness

One harness, mirroring `src/harness.tw`. The quantile interpolation matters:
`numpy.quantile`'s default linear method matches bobbin's `quantile_sorted`.

```python
import time
import numpy as np

def bench(body, inner=1, warmup=3, min_samples=30, min_total_ns=1_000_000_000,
          max_samples=1000, max_total_ns=30_000_000_000):
    for _ in range(warmup):
        body()
    samples, total = [], 0
    while True:
        t0 = time.perf_counter_ns()
        for _ in range(inner):
            body()
        took = time.perf_counter_ns() - t0
        if took >= 0:
            samples.append(took // inner)
            total += took
        if len(samples) >= max_samples or total >= max_total_ns:
            break
        if len(samples) >= min_samples and total >= min_total_ns:
            break
    xs = np.array(sorted(samples))
    q1, med, q3 = np.quantile(xs, [0.25, 0.5, 0.75])
    return {"median": med, "iqr": q3 - q1, "min": int(xs[0]), "n": len(xs),
            "outliers": int((xs > q3 + 1.5 * (q3 - q1)).sum())}
```

`auto_inner` is the same loop as bobbin's: grow `inner` geometrically until a
batch lasts a millisecond. Every body must consume its result (the returns
below exist for that), or a sufficiently clever runtime deletes the work.

### mlp

PyTorch. `dt` is one of `torch.float64`, `torch.float32`, `torch.bfloat16`;
inputs are drawn at float64 and cast once, as in the suite.

```python
import torch
torch.manual_seed(101)
X  = torch.randn(32, 784, dtype=torch.float64)
T  = torch.randn(32, 10, dtype=torch.float64)
W1 = torch.randn(784, 256, dtype=torch.float64) * 0.05
B1 = torch.zeros(256, dtype=torch.float64)
W2 = torch.randn(256, 10, dtype=torch.float64) * 0.05
B2 = torch.zeros(10, dtype=torch.float64)

def mlp_fwd(p):
    return torch.relu(p["X"] @ p["W1"] + p["B1"]) @ p["W2"] + p["B2"]

def mlp_fwdbwd(p):                     # p's weights have requires_grad=True
    for k in ("W1", "B1", "W2", "B2"):
        p[k].grad = None
    loss = ((mlp_fwd(p) - p["T"]) ** 2).mean()
    loss.backward()
    return sum(float(p[k].grad.sum()) for k in ("W1", "B1", "W2", "B2"))
```

NumPy, forward and the hand-written backward of the same loss:

```python
def mlp_fwd(p):
    return np.maximum(p["X"] @ p["W1"] + p["B1"], 0.0) @ p["W2"] + p["B2"]

def mlp_fwdbwd(p):
    Z = p["X"] @ p["W1"] + p["B1"]
    H = np.maximum(Z, 0.0)
    P = H @ p["W2"] + p["B2"]
    dP = 2.0 * (P - p["T"]) / P.size
    dW2, dB2 = H.T @ dP, dP.sum(0)
    dH = (dP @ p["W2"].T) * (Z > 0.0)
    dW1, dB1 = p["X"].T @ dH, dH.sum(0)
    return dW1.sum() + dB1.sum() + dW2.sum() + dB2.sum()
```

### conv2d

Both twill's `conv2d` and PyTorch's are cross-correlations, so no kernel flip
is needed anywhere. PyTorch wants a batch axis; it is added and removed at the
edges and is not part of the workload.

```python
import torch.nn.functional as F
torch.manual_seed(102)
CX = torch.randn(16, 64, 64, dtype=torch.float64)
CW = torch.randn(32, 16, 3, 3, dtype=torch.float64) * 0.1

def conv_fwd(p):
    return F.conv2d(p["X"].unsqueeze(0), p["W"]).squeeze(0)

def conv_fwdbwd(p):                    # X and W have requires_grad=True
    p["X"].grad = None; p["W"].grad = None
    conv_fwd(p).sum().backward()
    return float(p["X"].grad.sum() + p["W"].grad.sum())
```

NumPy forward, via a strided window view; the backward cell is absent, as
stated above.

```python
from numpy.lib.stride_tricks import sliding_window_view

def conv_fwd(p):                       # X [Cin, H, W], W [Cout, Cin, 3, 3]
    win = sliding_window_view(p["X"], (3, 3), axis=(1, 2))  # [Cin, 62, 62, 3, 3]
    return np.tensordot(p["W"], win, axes=([1, 2, 3], [0, 3, 4]))
```

### attention

```python
torch.manual_seed(103)
AX = torch.randn(128, 64, dtype=torch.float64)
Wq, Wk, Wv = (torch.randn(64, 64, dtype=torch.float64) * 0.125 for _ in range(3))

def attention_fwd(p):
    q, k, v = p["X"] @ p["Wq"], p["X"] @ p["Wk"], p["X"] @ p["Wv"]
    return torch.softmax((q @ k.T) / 8.0, dim=1) @ v
```

NumPy is the same four lines with a stable softmax written out, because NumPy
has none:

```python
def softmax_rows(s):
    e = np.exp(s - s.max(axis=1, keepdims=True))
    return e / e.sum(axis=1, keepdims=True)

def attention_fwd(p):
    q, k, v = p["X"] @ p["Wq"], p["X"] @ p["Wk"], p["X"] @ p["Wv"]
    return softmax_rows((q @ k.T) / 8.0) @ v
```

### softmax + cross-entropy

The logsumexp form, exactly as the suite and `std/loss` write it. Not
`F.cross_entropy`, which fuses and takes integer labels; the fused form may be
reported additionally, labelled.

```python
torch.manual_seed(104)
L = torch.randn(64, 1000, dtype=torch.float64)
Tgt = torch.eye(1000, dtype=torch.float64)[torch.randn(64, 1000).argmax(1)]

def softmax_xent(p):
    return (torch.logsumexp(p["L"], dim=1) - (p["L"] * p["T"]).sum(1)).mean()
```

NumPy, with logsumexp written out in its max-subtracted form, which is also
what twill's builtin computes:

```python
def softmax_xent(p):
    m = p["L"].max(axis=1, keepdims=True)
    lse = (m + np.log(np.exp(p["L"] - m).sum(axis=1, keepdims=True))).squeeze(1)
    return (lse - (p["L"] * p["T"]).sum(1)).mean()
```

### adam_step

The update written out, identical in all three implementations, applied to the
same fixed state every iteration (the benchmark measures the step, not the
trajectory). t = 10, lr = 0.001, b1 = 0.9, b2 = 0.999, eps = 1e-8.

```python
def adam_step(p, g, m, v, t=10, lr=1e-3, b1=0.9, b2=0.999, eps=1e-8):
    out = {}
    bc1, bc2 = 1.0 - b1 ** t, 1.0 - b2 ** t
    for k in p:
        m2 = b1 * m[k] + (1.0 - b1) * g[k]
        v2 = b2 * v[k] + (1.0 - b2) * g[k] * g[k]
        out[k] = (p[k] - lr * (m2 / bc1) / ((v2 / bc2) ** 0.5 + eps), m2, v2)
    return out
```

One honesty note this workload needs: real bf16 training keeps the optimiser
state and the master weights in f32 and rounds a bf16 copy of the weights, as
twill's dtype design specifies. The `adam_step/bf16` point instead runs the
whole update in bf16, in every implementation, because the point exists to
measure what halving every operand's width does to an elementwise, memory-bound
kernel. It is a bandwidth probe, not a recommended training configuration, and
a comparison that presents it as the latter is misusing it.

### eltwise_chain

```python
torch.manual_seed(106)
EX = torch.randn(16_777_216, dtype=torch.float64)

def chain(p):
    x = p["X"]
    return (torch.sqrt(torch.abs(torch.relu(x * 1.25 + 0.25) * x - 0.125)) + x).sum()
```

NumPy is the same line with `np.maximum(t, 0.0)` for relu. The constants are
exact in bf16, so every implementation at every dtype runs the same
arithmetic.

## Coverage

The cells that exist, and the ones that are absent with their reasons. An
absent cell is printed as absent in any published table; a table that quietly
drops a column is the start of cherry-picking.

| workload | twill | PyTorch | NumPy |
| --- | --- | --- | --- |
| `mlp_fwd` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `mlp_fwdbwd` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `conv2d_fwd` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `conv2d_fwdbwd` | f64 f32 bf16 | f64 f32 bf16 | absent: no autodiff, and a hand-rolled backward benchmarks the transcription |
| `attention_fwd` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `softmax_xent` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `adam_step` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |
| `eltwise_chain` | f64 f32 bf16 | f64 f32 bf16 | f64 f32 |

NumPy's bf16 column is absent throughout: NumPy has no bf16, and simulating
one changes the accumulation semantics being measured.

## The honesty rules

These are the rules that make a number from this suite publishable. They bind
the person publishing, which includes us, and especially includes runs where
twill loses.

1. **Same math, same shapes, same dtype, same discipline.** Every compared
   cell runs the listings above under the harness above. A deviation is named
   in the published table or the cell does not appear.
2. **Medians with spread, always.** A comparison quotes median and IQR for
   both sides, with sample counts. A single number with no spread is not a
   result and is not quoted, in prose, in a README, or anywhere else.
3. **No cherry-picking.** A published run is the whole suite: every workload,
   every dtype point, every implementation attempted, absences shown as
   absences. The suite is fixed precisely so that the set of things reported
   is not chosen after seeing the numbers.
4. **Same machine, same session, recorded.** Both sides of a comparison run
   on the same machine in the same power state, and the machine, thread
   counts, and library versions are recorded with the numbers. Numbers from
   different machines are never placed in one table.
5. **Inconclusive is a result.** A cell whose run was too noisy reports
   inconclusive, per bobbin's rules. It is not rerun until it says something
   nicer; reruns are reported as reruns.
6. **Unexplained improvements are investigated.** A speedup with no known
   cause is treated as a possible measurement bug first and a result second.
   An improvement that turns out to be the optimiser deleting the work is the
   canonical way benchmark suites die, and `keep` exists because of it.
7. **The suite survives the transition.** The same file runs on the Go
   bootstrap today and on the self-hosted runtime later, with baselines
   recorded on both sides, so a regression introduced by self-hosting is
   visible in one diff. The suite is not edited at the transition; that is
   rule 3 wearing its most tempting disguise.
8. **The claim stays inside the measurement.** This suite supports statements
   about these eight workloads at these shapes on CPU. It says nothing about
   GPUs, other shapes, or workloads not in it, and no published sentence gets
   to imply otherwise. The twill repository's `docs/gpu-feasibility.md` is
   the standing example of what measuring before claiming looks like; this
   suite exists so ML claims are held to the same bar.

## See also

- `suites/ml.tw`, the suite itself, which governs.
- `src/stats.tw` and `src/harness.tw`, the statistics and the protocol.
- `docs/needs.md` entries 14 to 16, the walls hit writing the suite.
- In the twill repository: `docs/dtypes.md` (the dtype design the axis
  measures) and `docs/gpu-feasibility.md` (the measured f64 against f32
  numbers that make the axis worth having).
