# What bobbin needs from twill

bobbin runs on twill 1.7.1: the tests pass and `examples/tensor_ops.tw` goes end
to end. This file is what is left, and what was closed: the language and runtime
features the source uses, with the file and function that needs each one, the
status against 1.7.1, and what bobbin does in the meantime.

It is a work queue for the language, not a complaint. Every entry was reached by
writing real code and hitting the wall, which is the only way a list like this is
worth anything.

Two of these are unusual for this list, because they are not about expressing a
program. A benchmarking tool needs the runtime to tell it things about itself,
and a language that cannot report its own allocation count cannot be profiled
from inside. Entries 1 and 2 are that.

The baseline is milestone 1 of `docs/self-hosting.md` in the twill repository:
`mode systems`, `I64`, `Str`, `Arr[T]`, `Dict[Str, V]`, `struct`, and
`read_file`.

## What bobbin could not measure anything without

### 1. A monotonic nanosecond clock

**Needs:** `mono_ns() -> I64`
**Used by:** `src/clock.tw` (`now_ns`, `probe`), and therefore every timing in
the repository
**Status:** landed in twill 1.6 as `mono_ns()`, and `src/clock.tw` calls it. The
resolution requirement below is *not* met on every platform. On Windows, on
twill 1.7.1, `clk.probe(100000)` reports an overhead of 254ns and a granularity
of 378,100ns: `mono_ns` is nanoseconds as a unit and not as a resolution, and a
378us tick is coarser than four of the five bodies in
`examples/tensor_ops.tw`. `src/harness.tw` refuses to report a median at or
below the measured granularity, and refuses one when the granularity could not
be measured at all, so the shortfall is visible rather than silent. This entry
stays open for the resolution alone.

One consequence is bobbin's and belongs here rather than in the language's
queue: `src/harness.tw` calls `clk.probe(64)`, and 64 consecutive reads on this
machine never straddle a tick, so `probe` returns a granularity of -1 and every
row of a real run carries "the clock's granularity could not be measured"
instead of the sharper "below one tick". The sample count is too low for a clock
this coarse. Not fixed here, because the number to raise it to is a function of
the tick and the tick is what this entry is about.

This is the whole tool. The requirements are in the header of `src/clock.tw` and
are repeated here because they are requirements on the runtime and not
preferences:

- **Monotonic.** A wall clock corrected mid-run produces a negative duration,
  and a negative duration in a sample set moves the median in the direction
  that looks like an improvement. `src/clock.tw` discards negative intervals
  defensively, which limits the damage and does not remove the need.
- **Nanoseconds.** A fast tensor op runs in microseconds. A millisecond clock
  measures it as zero, and a harness given zeros reports a median of zero and an
  IQR of zero, which reads as a perfectly stable, infinitely fast operation.
- **`I64`, not `F64`.** Nanoseconds since an arbitrary origin fit in an I64 for
  292 years. An F64 has 53 bits of mantissa and starts losing nanosecond
  resolution after about 104 days of uptime, silently, and the loss looks like
  quantisation in the timings.
- **Cheap, and non-allocating.** The call brackets the work; if it costs a
  comparable amount, the harness is measuring itself. `clk.probe` measures the
  overhead and the granularity so a result can say when this was violated, which
  is the most bobbin can do from outside.

If only one of these entries is ever implemented, this is the one.

### 2. Allocation and memory counters

**Needs:** `mem_counters_available() -> Bool`, `mem_allocs() -> I64`,
`mem_bytes() -> I64`, `mem_live_bytes() -> I64`, `mem_tensors() -> I64`
**Used by:** `src/memory.tw` (`read`), `src/harness.tw` (`run`)
**Status:** three of the four landed in twill 1.6 and still hold on 1.7.1.
`mem_counters_available()` returns true, and `mem_allocs`, `mem_bytes` and
`mem_live_bytes` all return real figures; a run of `src/harness.tw` with
`measure_memory` set prints `18 allocs/iter, 3KiB/iter` for an elementwise
multiply over a 256-element vector, which is a measurement and not a stub.
`mem_tensors` exists as a name and returns -1, which the runtime documents as
"not counted". `src/memory.tw` carries a second flag, `tensors_counted`, for
exactly that: subtracting two -1 sentinels gives 0, and zero tensors per
iteration is the answer someone tuning tensor code most wants to see. This entry
stays open for the tensor counter alone.

Four counters, and each answers a different question:

- `mem_allocs` is the one that catches a regression. Bytes move with input size
  and with the allocator's rounding; a count moves when the code allocates
  somewhere new, which is what changed.
- `mem_bytes` catches the other regression: the same number of allocations,
  each larger, which is what a wrong intermediate shape looks like.
- `mem_live_bytes` is the only one that can go down, and the only one that
  answers "does this fit".
- `mem_tensors` is twill-specific and the most useful of the four here. In
  tensor code one avoidable temporary per element is the difference between a
  benchmark that fits in cache and one that does not, and no general-purpose
  allocation count separates a tensor from a slice header. This is the one still
missing.

All must be cheap and must not themselves allocate. The `available` flag exists
because bobbin has to distinguish "zero allocations" from "cannot measure", and
printing a zero for the second is the most misleading thing a profiler can do.
Where the counters are absent, `mem.read()` returns `unavailable()` and every
reporter omits the memory columns entirely; where only the tensor count is
absent, the reporters print the other three and say the tensor count was not
counted.

### 3. A compiler barrier, or a guarantee there is nothing to barrier against

**Needs:** an operation the optimiser cannot see through, `black_box(x)`
**Used by:** `src/harness.tw` (`keep`)
**Status:** twill is interpreted and removes nothing, so this is not blocking
today. It becomes blocking the day it is not.

A benchmark body's result is discarded, and discarded work is work a compiler
may delete. Every serious benchmarking library has this and every one of them
added it after somebody published a number for an empty loop. `bench.keep` is a
named identity function so that when twill grows an optimiser there is exactly
one place to fix, and this entry is here so the fix is not forgotten between now
and then. Filed early on purpose: after the fact it is a retraction.

### 4. Writing files

**Needs:** `write_file(path: Str, contents: Str) -> Res[Unit, Str]`, and a
reader for what it wrote
**Used by:** `src/report.tw` (`render_baseline`), and any real runner
**Status:** delivered. twill 1.7.1 has `write_file(path, contents)` returning
`Res[Unit, Str]` and `read_file(path)` beside it, and both work: a write of
`"hello\n"` followed by a read returns `hello`. It also has `file_size`,
`read_file_at`, `mkdir_all`, `remove_file`, `remove_dir`, `remove_all`,
`rename`, `temp_dir`, `path_exists`, `path_is_dir`, `mtime` and the `path_*`
helpers, which is more than this entry asked for.

The remaining gap is bobbin's, not the language's. `src/report.tw`
(`render_baseline`) still only returns the text, nothing calls `write_file` with
it, and nothing parses a baseline back, so every run compares against an empty
baseline and reports every benchmark as new. The honest state is unblocked and
unwired. Wiring it is the next piece of work in this repository, and it needs a
parser for the TOML subset below rather than a new language feature.

`src/report.tw` renders the baseline in the same TOML subset spool reads, so
whichever of the two grows a parser first, the other can use it.

## Language features the source assumes

### 5. Function values as parameters and struct fields

**Used by:** `src/harness.tw` (`run`, `batch`, `auto_inner` all take
`body: fn(I64) -> F64`), `src/suite.tw` (`Case.body` is a stored function)
**Status:** delivered. A systems-mode function takes one as `fn(I64) -> F64` and
a struct field stores one; `src/suite.tw` compiles and `twill run
examples/tensor_ops.tw` calls five stored bodies through `Case.body`. The
spelling is the one this source already used.

There is no way to benchmark a piece of code without being handed it. `Case`
storing a function in a struct field is the stronger of the two asks, and
`src/suite.tw` is unwritable without it: a suite is a list of named pieces of
code, and that is what it is.

The same entry appears in loom's `docs/needs.md` as entry 3. Two independent
packages hitting it is worth something.

### 6. `break` and `continue`

**Used by:** `src/harness.tw` (`run`, the sampling loop)
**Status:** delivered by the language, not adopted by bobbin. Both `break` and
`continue` work in a systems-mode `while` on twill 1.7.1. `src/harness.tw` still
carries the `done` flag and the comment at line 124 still says twill has no
`break`, which is now wrong.

The sampling loop runs until a minimum sample count and a minimum elapsed time
are both met, or a ceiling is hit. That is four exit conditions checked at the
bottom of a loop, which is exactly what `break` is for. bobbin uses a `done`
flag, which is readable enough here and would not be in a loop with real work
after the check. Replacing it is a change to a loop that every timing in the
repository goes through, so it is worth doing on its own and not in passing.

### 7. twill's terminal layer, reachable from a package

**Needs:** `src/term/` reachable from a package
**Used by:** `src/report.tw`, which now calls it
**Status:** RESOLVED. twill's terminal modules were made import-portable.

The premise here was wrong, and it was the twill side that fixed it. twill's
`src/term/` and `src/cli/` modules were converted to import each other by a
path relative to the importing file rather than to the working directory, and
`resolveImport` tries the importer-relative path first. So a package that
vendors twill under `twill_modules/` can reach the terminal layer with
`import "../twill_modules/twill/src/term/caps.tw"`, exactly the way weft reaches
`chart` and `box`. bobbin now does: `src/report.tw` detects capabilities once
and lights each verdict in its colour, the regression red and the improvement
mint, dropping to plain text the moment the output is piped. The `!!` and `++`
markers stay underneath the colour so neither reading depends on the other.

This was loom's 8 as well; both are satisfied by the one portability change.

## Not blocking, but the source is worse without them

### 8. A test runner

**Would improve:** `tests/`
**Status:** delivered. `twill test tests` collects the five `*_test.tw` files,
runs each and reports once: "5 file(s): 5 passed, 0 failed". CI runs exactly
that command.

`tests/harness.tw` is still a hand-rolled counter and is still the third
byte-identical copy of that file in the ecosystem, after spool's and loom's.
`twill test` reports per file, so the per-assertion names those three files
provide are not yet something the runner supplies. Deleting them is a separate
question from having a runner, and this entry asked for the runner.

### 9. A generic sort, or a comparison-function parameter

**Would improve:** `src/stats.tw` (`sorted`), `src/baseline.tw` (`put`)
**Status:** open. twill 1.7.1 has a builtin `sort`, but on a list it requires
every element to be a string: `sort([3, 1, 2])` on an `Arr[I64]` fails with
"sort on a list expects every element to be a string". Neither of bobbin's two
insertion sorts can use it. Function parameters landed (entry 5), so a
comparison-function form is now expressible; nothing exposes one.

Two more insertion sorts, on top of spool's four and loom's one. Seven. The one
in `src/stats.tw` is also the hot path of the whole tool: every summary sorts its
samples, and an insertion sort over a thousand samples is a million comparisons
in an interpreter. A builtin sort over `Arr[I64]` would be the single largest
speedup available to this repository.

### 10. `Res[T, E]` and `Opt[T]`

**Used by:** `src/baseline.tw` (`find`), `src/suite.tw` (`validate`)
**Status:** done (2026-08, on twill 1.7).

`find` returned `-1` for an absent entry, which the paragraph below called the
usual bad sentinel: `-1` is a valid I64, nothing forced a caller to test it, and
both callers had to remember. It returns `Opt[I64]` now, so there is no index to
read without saying which branch you are in. `put` keeps using the index to
replace in place and `compare` takes the `None` arm to build its `New` verdict,
which is why the index and not the entry is what comes back.

`validate` returned an empty string for success and returns `Res[Unit, Str]`.

*Previously:* "Sentinel returns throughout. `find` returning `-1` is the usual
bad one: it is a valid I64, nothing forces a caller to check it, and an
unchecked `-1` indexes from the end of an array in many languages and would here
too if `Arr` allowed it."

### 11. Sum types and `match`

**Would improve:** `src/baseline.tw` (`verdict_name`, `compare`),
`src/report.tw` (`human_comparison`)
**Status:** done, and this status line was stale for two releases. twill shipped
`enum` with payloads, `match` and enforced exhaustiveness in 1.6 and bobbin
adopted it at the same time.

`src/baseline.tw` declares
`enum Verdict { Pass, Regressed, Improved, Inconclusive, New, Missing }`, and
all three dispatches over it -- `verdict_name`, and `verdict_style` and
`verdict_arrow` under `human_comparison` -- are exhaustive matches. A seventh
verdict is now a check-time error naming the missing arm in each of them, which
is exactly what the paragraph below asked for.

*Previously, and now describing nothing:* "Six verdicts as I64 constants and two
if-chains over them, in different files. Adding a seventh verdict compiles and
silently falls through to \"missing\" in one place and to the default marker in
the other."

### 12. Integer formatting with a thousands separator, and string padding

**Would improve:** `src/report.tw` (`pad_left`, `pad_right`), `src/clock.tw`
(`fixed`, `pad_zero`)
**Status:** open. `str(x)` exists and produces a shortest round-trip form.
Neither a padding builtin nor a format builtin does: `pad_left` and `format` are
both unknown names on twill 1.7.1, so the four helpers stay hand-rolled.

Four hand-rolled formatting helpers, one of which reimplements fixed-point
decimal output character by character. Column alignment is not a nicety in a
benchmark table: varying width is what hides the differences the table exists to
show. loom has the same `fixed` function, copied, which is docs/needs.md entry
8 wearing a different hat.

### 13. A statistical note, not a language request

**Status:** not a request. Recorded so the decision is not relitigated silently.

There is no significance test in `src/baseline.tw` and that is deliberate. A
t-test assumes a normality timings do not have. A Mann-Whitney U on a few hundred
samples reports significance for differences far below what anyone would act on,
because with enough samples everything is significant. The question a regression
gate is asked is not "is this difference real" but "is this difference big
enough to care about", and that is a threshold question, so bobbin uses two
thresholds and says which one was not met.

## Hit while writing the ML workload suite

`suites/ml.tw` is the fixed ML workload suite and `docs/ml-workloads.md` is
its methodology. Writing it hit three more walls, found the same way as the
thirteen above: by writing the real code.

### 14. dtype names, and a dtype a program can pass

**Needs:** NEEDS-110 in the twill repository (`bf16` as a name,
`zeros(shape, bf16)`, `x.to(dt)`), and one thing more than it asks for: a
dtype usable in ordinary argument position
**Used by:** `suites/ml.tw` (`to32`, `tob16`, every input set, every body)
**Status:** NEEDS-110 shipped; the extra ask is still open.

The spelling parses and runs on twill 1.7.1. `zeros(4, f32)` builds a tensor
that prints `dtype=f32`, `x.to(bf16)` prints `dtype=bf16`, `twill check
suites/ml.tw` is clean and `twill run suites/ml.tw` exits 0. `f64` is the
untagged default rather than one of the contextual names: `zeros(4, f64)` fails
with "zeros expects a tensor/number", while `x.to(f64)` is accepted and drops
the dtype tag. `suites/ml.tw` never writes `zeros(shape, f64)`, so this does not
bite it, and it is worth knowing before someone writes the axis as three
symmetrical cases.

The wall is the part past the design, and it is where it was. NEEDS-110 makes the seven dtype names
contextual: `f32` reads as a dtype only in the dtype argument of a constructor
or of `.to`, and stays an ordinary identifier everywhere else. So
`inputs(f32)` is an unbound identifier, a function cannot take the axis as a
parameter, and the axis cannot be written once. The suite therefore contains
two cast helpers instead of one, three input records per workload instead of a
loop, and twenty-four benchmark bodies instead of eight. For one file that is
verbose and survivable. For the ecosystem's benchmark surface, where every
suite will want this exact axis, a dtype that is a value, as it is in numpy,
deletes two thirds of every one of those files.

### 15. packed buffers, before the dtype axis measures anything

**Needs:** NEEDS-111 in the twill repository, the packed byte-addressable
buffer
**Used by:** the interpretation of every `/f32` and `/bf16` point in
`suites/ml.tw`, `eltwise_chain` most of all
**Status:** open in the twill repository, and now measured rather than assumed.
On twill 1.7.1, `mem_live_bytes` across `zeros(100000, f32)` moves 1,606,008
bytes and across `zeros(100000, bf16)` moves 1,606,344: about 16 bytes per
element either way, whatever that 16 is made of, and identical across the two
dtypes: the narrow dtype buys back no memory at all. Not a new request; a
dependency recorded here so the suite's first run is not misread.

Until the layout lands, a bf16 element occupies 64 bits like everything else,
all three dtype points move identical bytes, and the narrow points do strictly
more work because they round on store. The f32 and bf16 columns of the first
run will come back *slower* than f64, and that is the correct result: it
measures the cost of dtype semantics, not the value of dtypes. The bandwidth
question the axis exists for is only answerable after NEEDS-111, and the
before-and-after pair of baselines across that landing is the measurement of
what the layout bought. `docs/ml-workloads.md` states the same rule for
anyone publishing.

### 16. the mode boundary, written down

**Used by:** all of `suites/ml.tw`
**Status:** unstated on both sides of the boundary.

The suite is `mode systems`, because the harness it drives is. It also stores
tensors in records, defines untyped functions over them, calls `grad`, and
imports `std/optim` for the optimiser step. Every one of those is
numeric-mode surface inside a systems-mode file. `examples/tensor_ops.tw`
already does the smaller half of this, calling tensor builtins from typed
systems-mode bodies, so the pattern is load-bearing in two files now and
nothing anywhere says it is legal.

What is needed is not a feature but a statement: what a systems-mode file may
do with numeric values, numeric functions and numeric `std/` modules, spelled
out in the self-hosting design. The alternative reading, that it may do none
of it, unwrites both files, and the time to learn which reading is true is
before the first runtime exists rather than the week it appears.
