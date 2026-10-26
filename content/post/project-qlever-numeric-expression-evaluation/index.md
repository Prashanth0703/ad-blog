---
title: "Optimizing numeric expressions in QLever"
date: 2026-10-26T12:00:00+05:30
author: "Prashanth Premakumar"
authorAvatar: "img/author_pp.jpeg"
tags: ["QLever", "SPARQL", "C++", "Optimization", "Benchmarking"]
categories: ["project"]
draft: true
---

A single addition is cheap. Evaluating it over a SPARQL result column can involve
much more work: finding inputs, interpreting datatypes, traversing generators,
and constructing results. This project investigates how much of that surrounding
work QLever can avoid while preserving its existing expression behavior.

<!--more-->

## Content

- [Why optimize an addition?](#why-optimize-an-addition)
- [The original path: more than arithmetic](#the-original-path-more-than-arithmetic)
- [First remove generator overhead](#first-remove-generator-overhead)
- [Then specialize homogeneous operands](#then-specialize-homogeneous-operands)
- [Mostly homogeneous data: keep the common row small](#mostly-homogeneous-data-keep-the-common-row-small)
- [Extending indexed execution to N-ary expressions](#extending-indexed-execution-to-n-ary-expressions)
- [Avoiding doomed conversions, and supporting dates](#avoiding-doomed-conversions-and-supporting-dates)
- [Why the classifier looks more complicated than necessary](#why-the-classifier-looks-more-complicated-than-necessary)
- [Controlled evaluation](#controlled-evaluation)
- [Where execution time goes now](#where-execution-time-goes-now)
- [Limits and remaining opportunities](#limits-and-remaining-opportunities)
- [Conclusion](#conclusion)

## Why optimize an addition?

Consider the expression in this query:

```sparql
SELECT ?x ?y ?sum WHERE {
  VALUES (?x ?y) { (1 10) (2 20) (3 30) }
  BIND(?x + ?y AS ?sum)
}
```

The small example is easy to evaluate. On a large intermediate result, however,
the same expression may run millions of times. Both columns might contain only
integers, yet a general evaluator still needs to support doubles, booleans,
undefined values, and operations involving dates. The question is whether that
flexibility has to be paid for on every row.

In this student project at the University of Freiburg, supervised by Johannes
Kalmbach, I investigated numeric expression evaluation in QLever. The work
progressed from a batching prototype to reusable binary and N-ary evaluation,
homogeneous and speculative numeric paths, and extensions for undefined inputs
and dates. The central questions are where execution time went originally,
which work the optimizations remove, and what remains afterward.

## The original path: more than arithmetic

QLever stores query results in tables of encoded values called Ids. An Id's tag
identifies its datatype; integer and double payloads can be accessed without a
vocabulary lookup. An expression's child result can also be a constant, a
variable referring to an input column, or another supported representation.

The original arithmetic evaluator used the generic `NaryExpression`
infrastructure. After evaluating its children, it created operand generators
that supplied values to the operation. Value getters interpreted the inputs.
A numeric getter returned a variant containing an integer, a double, or a
non-numeric marker; addition's date-aware getter also supported dates. The
operation then dispatched on those alternatives, performed the arithmetic,
encoded the result as an Id, and appended it to an output vector.

That is useful separation of responsibilities. It also places several layers
between a column element and its addition. For homogeneous integers, the answer
to the type question is identical on every row, but the generic path repeats
getter and variant work. Generator traversal adds another kind of indirection.

The type erasure was deliberate: exposing every combination of input types to
the compiler can make template instantiation prohibitively expensive. Removing
runtime overhead therefore also required respecting compilation constraints.

### Why the first batching prototype did not help

My first prototype converted batches of Ids into intermediate `NumericValue`
vectors and then added them. Early experiments reported that it could be slower
than the original implementation. Grouping work into batches had not eliminated
the conversion and variant processing, and materializing another representation
introduced additional work.

A standalone benchmark, constructed like the expression unit tests, made that
cost visible without involving the server, planner, or result export. The
historical profiles show how the investigation developed:

| Prototype profile | Important inclusive profile weights | Finding |
|---|---|---|
| Initial batching prototype (July 2) | Addition evaluator 94.82%; nested `getNumericBatch` 75.49% | Investigate conversion/materialization before the arithmetic. |
| Integer-specialized conversion (July 7) | Conversion subtree 47.50%; separate variant frame 27.29%; `isAllInt` 5.36% | Specializing conversion still leaves intermediate representation costs. |
| DBLP query (July 14) | Addition evaluator 52.41%; nested multiplication 51.42%; nested YEAR evaluator 20.18% | Much of the apparent addition cost belongs to child expressions. |

Open the complete interactive profiles in new tabs:
<a href="img/flamegraph-batched-add-2026-07-02.svg" target="_blank" rel="noopener">Batch conversion dominates the first prototype</a>,
<a href="img/flamegraph-integer-conversion-2026-07-07.svg" target="_blank" rel="noopener">Integer conversion still leaves variant overhead</a>, and
<a href="img/flamegraph-dblp-direct-integer-2026-07-14.svg" target="_blank" rel="noopener">DBLP: child expressions dominate the addition frame</a>.
Click frames to zoom and search for the named functions.

These are historical prototype profiles, not before/after measurements of the
final implementation. Their percentages are inclusive: parent and child widths
overlap. In the integer-conversion graph, `getNumericBatch` and `getNumericBatchInt` both show
47.50% because they cover the same nested subtree. They must not be counted twice.
A larger percentage across two graphs also does not imply a larger absolute cost.

The DBLP graph illustrates another trap: the addition evaluator includes the
work of its multiplication children. Its 52.41% width is not the cost of the
addition instruction. Likewise, an export-related ancestor can include lazy
query computation; its width does not isolate serialization. This motivated
reusable improvements beyond a dedicated addition expression.

An <a href="img/flamegraph-initial-unsymbolized-2026-06-25.svg" target="_blank" rel="noopener">initial server profile with unresolved symbols</a>
contains many unresolved server frames and is less useful for named-function
attribution. Exact binary revisions, event settings, and raw recordings are not
embedded in the surviving SVGs, so they provide development context rather than
controlled final results.

## First remove generator overhead

The first integrated change, [PR #3205](https://github.com/ad-freiburg/qlever/pull/3205),
introduced `BinaryExpression`. It converts variables to non-owning column spans,
expands interval sets when necessary, and evaluates vector or constant operands
with indexed access. The operation and its value getters are template parameters,
so the infrastructure is reusable rather than specific to addition.
Addition, subtraction, multiplication, division, and power use this infrastructure.

At this stage, getters still interpret values and operations still handle
variants. The improvement is a simpler route to each row. A constant's getter
can additionally run once per expression evaluation, rather than repeatedly for
every row. Constant–constant input produces a scalar result.

The output still uses QLever's memory-limited allocator. Cancellation checks
remain present, grouped at chunk boundaries. Common child storage, cache keys,
and determinism handling are shared with N-ary expressions. This is important
for integration: a faster inner loop must continue to work as an ordinary
expression in the query engine.

## Then specialize homogeneous operands

Removing generators does not remove generic type handling. The next step,
[PR #3366](https://github.com/ad-freiburg/qlever/pull/3366), classifies supported
Id operands before evaluation. If each operand is homogeneous, the evaluator
dispatches once to primitive C++ types and runs a typed loop.

The idea can be summarized with schematic pseudocode:

```text
classify each operand
if every operand has a supported homogeneous type:
    select the typed operation once
    for each row:
        extract known primitive values
        apply operation and encode the result
else:
    consider speculative execution or use generic evaluation
```

The operands need not have the same type: an integer column and a double column
can select the corresponding mixed primitive combination. Scalar inputs are
classified directly, and their primitive value can be extracted once. Unsupported
representations retain the generic path.

The fast path does not discard result construction. Operations wrapped in
`MakeNumericExpression` are unwrapped to `NumericIdWrapper`, which avoids
per-row variant visits while preserving the result-encoding policy, including
whether non-finite results become UNDEF. Addition and subtraction reuse their
existing primitive overloads. Classification is included in complete-expression
timings; measuring only the typed loop would overstate the benefit.

## Mostly homogeneous data: keep the common row small

An all-or-nothing fast path is fragile: one double in an otherwise integer column
would send the entire expression through generic evaluation.
[PR #3417](https://github.com/ad-freiburg/qlever/pull/3417) instead records a
preferred datatype for speculative execution.

In the final implementation, “majority” means the unique most frequent datatype,
not necessarily more than half the rows. Selection considers every datatype. A
column containing three doubles, two integers, and many unsupported values must
not be classified as predominantly double. Ties provide no preferred type.

If every operand has an eligible preferred type, matching rows use primitive
operations. Exceptional binary rows enter a separate slow function. Other
integer/double combinations are handled directly there; remaining cases retain
the normal getters and generic operation. Keeping this machinery outside the
common loop helps avoid burdening every matching row with the full fallback.

This is guarded execution, not unchecked arithmetic on arbitrary bit patterns
followed by repair. Tags are checked before primitive extraction. Booleans remain
accepted through their existing getter conversion, even though they have no
homogeneous primitive fast path. A heavily mixed column may still benefit much
less, especially when ties prevent speculative selection.

## Extending indexed execution to N-ary expressions

[PR #3604](https://github.com/ad-freiburg/qlever/pull/3604) applies the same direction
to the generic N-ary evaluator. After existing specialized-function handling,
operands are converted to indexable vectors, spans, or constants. One indexed
getter is associated with each operand, and an indexed loop invokes the operation.
Eligible numeric expressions can first try the shared homogeneous or speculative
helpers. In production, these numeric N-ary fast paths apply to the unary numeric
functions, such as unary minus, `ABS`, `ROUND`, and `LOG`. Other N-ary expressions,
including string, language, geo, and conditional expressions, receive the indexed
loop improvement rather than a numeric specialization. This is the distinction
made in the [final PR description](https://github.com/ad-freiburg/qlever/pull/3604).

This is more involved than adding another argument to a binary loop. An operation
with N arguments can encounter many combinations of representations. Compile-time
eligibility checks restrict numeric specialization to supported getters and Id
operands; date combinations are instantiated only for date-aware getters. The
generic fallback still supports other result types and their local-vocabulary
handling. The cheaper-compilation alternative remains available.

The implementation therefore shares conversion and evaluation machinery without
pretending that all SPARQL values are primitive numbers. Compiler compatibility
also mattered: named temporaries were necessary to work around Clang 16/17
code-generation failures. The evaluation below uses a synthetic three-argument
sum to probe this infrastructure; it is not presented as a SPARQL built-in.

## Avoiding doomed conversions, and supporting dates

### Direct UNDEF where semantics permit it

Sometimes the datatype already establishes the result. For an eligible numeric
operation, an operand rejected by the numeric getter can only produce UNDEF.
[PR #3708](https://github.com/ad-freiburg/qlever/pull/3708) lets evaluation return
that result without constructing numeric variants and invoking the operation.

Eligibility is explicit. A function must opt into the guarantee, and the check
must match its value getter. Booleans remain valid because numeric getters
convert them to integers. Date-aware addition/subtraction need a different rule:
a date is not a primitive number, but it can participate in valid arithmetic.
The initial UNDEF change therefore left date-aware addition and subtraction on
their existing fallback; the subsequent date work extended the rejection rule
while allowing date operands. The numeric-only shortcut covers the unary numeric
functions, multiplication, division, and power.

Classification also records whether any datatype is rejected by the getter.
When it proves all values are accepted, evaluation can omit redundant rejection
checks. Otherwise, direct-UNDEF checks add work to valid rows while saving work
on invalid ones. Fully numeric controls and different invalid-value mixtures
are therefore necessary to assess the tradeoff.

### Homogeneous dates and durations

The date follow-up, [PR #3737](https://github.com/ad-freiburg/qlever/pull/3737),
shares the numeric machinery rather than maintaining a second evaluator. For
eligible date-aware getters, classification can select `DateYearOrDuration` as
the typed representation. The existing date operation still determines whether
particular dates, dateTimes, durations, and their combinations are compatible.

A shared Id datatype does not make all date payloads semantically interchangeable.
The fast path extracts the date representation and calls existing typed arithmetic;
it does not replace timezone or duration rules with raw integer subtraction.
Speculative execution can likewise handle eligible date rows while preserving
fallback behavior for outliers.

## Why the classifier looks more complicated than necessary

The final classifier processes blocks of 32 Ids. It extracts their datatype tags
and ORs the XOR of each tag with the first. A zero result establishes homogeneity,
allowing one histogram update for the entire block. Mixed blocks try a two-type
counting reduction, validate that those types cover the block, and otherwise use
a general histogram. Incomplete blocks are handled separately.

Why not simply stop when a mismatch is found? An early-exit variant was tested
during development and rejected after it regressed homogeneous input. Regular
reductions give the compiler a different optimization opportunity from a
row-dependent exit. This historical experiment is separate from the current
compiler evidence: GCC 13.3 reports 16-byte vectorization for tag extraction,
the OR/XOR reduction, and two-type counting. The general histogram loop does
not vectorize.

Assembly contains packed shifts, XORs, ORs, comparisons, and additions in the
classifier, and SIMD instructions in the typed evaluation loops. These findings
apply to this compiler and target. They do not show that explicit SIMD would
necessarily be faster, nor establish a bandwidth bottleneck.

## Controlled evaluation

The three-stage experiment compares the original production evaluator, indexed
generic binary evaluation, and the final project branch:

| Revision | Role |
|---|---|
| `6358e102802b8c192630814fb8340d2e8dee2c49` | Before PR #3205; original generic evaluator |
| `9290ebe087ac7a8c20253fe4b8ca40f7ca893949` | Before PR #3366; indexed, generic binary evaluation |
| `f86881abd8758132017a2d9a747b0fd645dcd79d` | Final project branch including date handling |

All three builds use GCC 13.3, C++20, Release optimization (`-O3 -DNDEBUG`),
no LTO, and a common copied dependency cache on a Ryzen 7 3700X. No native CPU
flag was added. Source exports and builds are independent of the development
checkout. The endpoints span 216 commits, including unrelated changes, and
common dependency versions differ from some historical pins. Endpoint speedups
therefore describe the complete comparison, not a causal contribution per PR.

Each workload has 100,000 rows and 50 expression evaluations per timed batch.
Ten independent processes per revision validate every output row during three
warmups. Data generation is outside timing; child evaluation, classification,
allocation, result construction, and destruction are inside. A compiler memory
barrier retains results. Workloads use a fixed seed, revisions alternate order,
and runs are pinned to CPU 2. The host is shared, its existing powersave governor
and boost remain unchanged, and the SMT sibling is not reserved.

### Three stages of evaluation

The table reports median milliseconds per batch and medians of paired overall
speedup ratios. Every mixture uses a fixed shuffled exception pattern. The full
comparison has 300 timed samples, with ten samples for each workload/revision.
Raw ranges are retained; occasional timing excursions remain on the shared host.

| Workload | Original ms | Indexed generic ms | Final ms | Overall speedup |
|---|---:|---:|---:|---:|
| Integer vector-vector | 109.454 | 73.114 | 6.643 | 16.47× |
| Double vector-vector | 110.575 | 72.431 | 4.853 | 22.99× |
| Integer vector-constant | 97.302 | 47.196 | 3.847 | 25.29× |
| Double vector-constant | 98.858 | 45.375 | 3.249 | 30.42× |
| 90% int / 10% double | 110.207 | 73.343 | 14.627 | 7.53× |
| 50% int / 50% double | 111.456 | 74.502 | 78.845 | 1.41× |
| 50% int / 50% UNDEF | 110.523 | 73.450 | 52.486 | 2.10× |
| Ternary integer | 147.738 | 146.041 | 9.439 | 15.66× |
| Ternary 90% int / 10% double | 147.292 | 145.958 | 26.327 | 5.60× |
| Ternary 50% int / 50% double | 147.628 | 146.395 | 116.629 | 1.26× |

The intermediate stage makes the infrastructure contribution visible. Integer
vector-vector addition improves from 109.45 to 73.11 ms before numeric fast paths
are introduced. Vector-constant addition benefits more, from 97.30 to 47.20 ms.
The homogeneous final path then removes much of the remaining generic handling.
As expected for a binary-only infrastructure change, the intermediate ternary
results stay close to the original evaluator.

The 50/50 numeric case exposes a cost hidden by an endpoint-only comparison:
final evaluation is about 5.8% slower than indexed generic evaluation, despite
still beating the original implementation. There is no unique preferred type,
so classification adds work before generic fallback. Historical revisions also
contain unrelated changes, so these stages do not isolate every PR individually.

The synthetic ternary sum reads the left column twice. It probes infrastructure
with a specific operand-reuse pattern, not the distribution of arbitrary N-ary
queries. The large homogeneous gains must not be generalized to all mixtures.

### Focused date and UNDEF controls

To narrow attribution for the extensions, a second experiment uses the same final
framework and operation on both sides. It toggles date dispatch or direct-UNDEF
eligibility in benchmark-only types; no production implementation is edited.
There are 260 timed samples across 26 variants. Each process validates all rows
before timing, using 100k rows, 50 evaluations, and the same affinity and compiler.
These timers measure core evaluation, including classification, allocation and
result destruction, but exclude child-expression evaluation. They are separate
from the complete-expression table above and are not reconstructed PR timings.

For date subtraction, the disabled control preserves numeric fast-path eligibility
and the date-aware getter's conversion semantics, but excludes dates from typed
dispatch. Both variants use the existing typed date subtraction and leave the
direct-UNDEF shortcut disabled. Expected date/duration results are constructed
independently as a one-day duration; the dateTime inputs include differing timezone
offsets. The date data repeats small sets of values.

| Workload | Disabled ms | Enabled ms | Paired speedup |
|---|---:|---:|---:|
| Date subtraction, vector-vector | 253.867 | 182.991 | 1.38× |
| Date subtraction, vector-constant | 224.315 | 182.758 | 1.23× |
| dateTime subtraction, vector-vector | 255.486 | 185.694 | 1.38× |
| Duration subtraction, vector-vector | 208.117 | 105.392 | 1.98× |
| Date subtraction, 30% UNDEF | 201.787 | 153.942 | 1.31× |
| Integer subtraction control | 6.618 | 6.629 | 1.00× |
| Double subtraction control | 4.866 | 4.869 | 1.00× |

Date and dateTime vector subtraction improve by about 1.38×, while duration
subtraction improves by about 1.98×. Numeric controls are nearly unchanged in this
experiment. These results concern these operations and distributions, not every
date-related SPARQL expression.

The [merged date/duration PR](https://github.com/ad-freiburg/qlever/pull/3737)
also reports measurements on Wikidata-sized inputs:

| Reported workload | Input size | Before | After |
|---|---:|---:|---:|
| `?death - ?birth` | 3.7 million humans with both dates | 0.39 s | 0.34 s |
| `?birth + "P30D"^^xsd:dayTimeDuration` | 8.1 million dates | 0.31 s | 0.26 s |

These figures come from the merged PR description; they were not rerun in the
local campaign. Its description does not provide the full environment and timing
boundaries, so the absolute times are not directly comparable to our core timers.
The same description reports gains and mixture-dependent regressions, consistent
with evaluating the optimization across more than homogeneous inputs.

The UNDEF experiment uses multiplication with standard numeric getters and
identical arithmetic, toggling only the function's direct-UNDEF opt-in. Exceptions
are shuffled; the tied case has equal integer and double counts and cannot select
a unique preferred type. The boolean case checks that accepted boolean conversion
is retained while undefined rows avoid unnecessary work.

| Workload | Disabled ms | Enabled ms | Paired speedup |
|---|---:|---:|---:|
| 70% integer / 30% UNDEF | 33.767 | 24.119 | 1.40× |
| 70% double / 30% UNDEF | 33.477 | 24.173 | 1.39× |
| 35% integer / 35% double / 30% UNDEF | 81.371 | 64.346 | 1.27× |
| 70% boolean / 30% UNDEF | 78.563 | 59.713 | 1.32× |
| All-integer control | 8.435 | 8.404 | 1.00× |
| All-double control | 6.656 | 6.791 | 0.98× |

The shortcut helps all four tested mixtures. It does not imply a free check in
every context: the all-double control's median is about 2% slower with the opt-in,
while the integer control is close. Small effects can depend on code generation
and shared-host variation; this result is reported rather than called universally
regression-free. Broader distributions would be needed to establish their impact.

## Where execution time goes now

Host policy currently denies perf events, so no security settings were changed.
Separate executables linked from the optimized objects enable GNU user-space
CPU sampling at 100 Hz. Approximately 1,000–1,200 executable-text samples per
profile provide a flat function-region view, not an instrumented call graph.
Shared-library costs outside the executable text, including allocator/libc work,
are not represented in these shares.

Default gprof output misattributed some coroutine code to neighboring symbols.
The analysis therefore resolves raw histogram bins against exact ELF symbol
boundaries and retains unresolved padding rather than inventing attribution.
The small uncertainty from four-byte bins and sampling remains; profiles and
wall-clock measurements are collected in separate runs.

| Workload | Old getter share | Old operation coroutine share | Final classifier share | Final typed-loop share |
|---|---:|---:|---:|---:|
| Integer vector-vector | 60.7% | 16.1% | 33.9% | 65.4% |
| Double vector-vector | 59.5% | 16.1% | 41.5% | 57.9% |
| Integer vector-constant | 46.7% | 20.7% | 27.9% | 71.9% |
| Ternary integer (left + right + left) | 65.6% | 14.0% | 31.2% | 68.4% |

For integer vector-vector addition, the original getter alone accounts for about
60.7% of executable-text profile weight, with another 16.1% in the operation
coroutine. After optimization, classification and typed evaluation dominate.
The typed loop includes loads, primitive decoding, arithmetic, Id encoding,
and stores; its whole width cannot be assigned to the arithmetic instruction.

Separate classifier-only runs take about 1.93 ms per integer vector-vector batch.
They classify two columns with numeric-only classification; production addition
also enables date recognition. Those component runs are therefore corroborating
evidence, not an exact additive decomposition. A larger relative classification
share after optimization does not mean classification became slower.

The final 50/50 mixed numeric profile still places about 61% in the generic
getter and about 4.4% in classification. This explains why eliminating the
homogeneous case's dispatch does not remove the same cost from all inputs.

## Limits and remaining opportunities

The evaluation is deliberately concentrated on expression execution. Query
planning, joins, storage access, and export can dominate complete queries, so
expression speedups must not be quoted as application speedups. The historical
DBLP profiles and the Wikidata measurements reported in the merged date PR provide
larger-workload context, but neither set was rerun in the local campaign.

The numeric payloads repeat values modulo 1,000; these bounded inputs avoid
overflow and do not constitute an exhaustive
semantic test suite. Existing numeric/date helper tests passed, including date
classification, block boundaries, and speculative outliers. The full SPARQL suite
was not rerun. Compiler, cache-size, clustering, and realistic distribution
sensitivity also remain limits of these measurements.

The largest remaining homogeneous region is the typed loop; tied mixtures still
spend substantial time in generic getters. Possible further work includes
classification reuse for repeated operands or a better treatment of those mixed
inputs. Neither is automatically a safe improvement: representation semantics,
cancellation, extra checks, and compiler behavior all matter. No additional
production optimization is claimed without a controlled gain and correctness
validation.

## Conclusion

The useful change was to move repeated decisions out of the common row. Indexed
evaluation removed generator traversal; homogeneous classification enabled one
typed dispatch; speculative execution kept exceptional rows from forcing every
row through generic handling. Operation-specific UNDEF checks and shared date
handling extended the approach without inventing new arithmetic semantics.

The measurements show substantial homogeneous expression improvements and much
smaller gains for tied mixtures. Profiling explains that difference: the original
work was dominated by generic getters and generator/variant machinery, while
the homogeneous final path spends its sampled time mainly in classification and
typed result production. Understanding that shift is more informative than a
single headline speedup, and identifies where further work would have to help.
