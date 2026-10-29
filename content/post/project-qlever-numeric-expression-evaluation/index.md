---
title: "Optimizing SPARQL Expression Evaluation in QLever"
date: 2026-10-26T12:00:00+05:30
author: "Prashanth Premakumar"
authorAvatar: "img/author_pp.jpeg"
tags: ["QLever", "SPARQL", "C++", "Optimization", "Benchmarking"]
categories: ["project"]
draft: true
---

QLever is a database engine for querying knowledge graphs. A knowledge graph
represents entities and their relationships as facts. For example, a fact can
state that a particular person was born in a particular city. QLever works with
RDF, the Resource Description Framework, which represents such facts as triples:
a subject, a predicate, and an object. In this example, the person is the subject,
“was born in” is the predicate, and the city is the object. SPARQL is a query
language for RDF data.

A SPARQL query can also calculate values. For example, the expression `?x + ?y`
adds the values assigned to two variables for each result row. The addition
itself is cheap, but QLever must also identify the input datatypes, obtain their
numeric values, and construct the result. When an expression runs over millions
of rows, this surrounding work can become expensive.

In this project, I implemented faster evaluation paths that reduce this overhead.
The experiments show the largest gains when each input column contains a single
datatype. Mixed columns benefit less, and some cases become slightly slower than
an intermediate implementation. These measurements concern expression evaluation,
not complete query execution. This post explains the original evaluator, the
changes, and the evidence for those results.

<!--more-->

## Content

- [Introduction and motivation](#introduction-and-motivation)
  - [Why optimize an addition?](#why-optimize-an-addition)
  - [The original evaluation path](#the-original-evaluation-path)
- [Implementation](#implementation)
  - [Lessons from the batching prototype](#lessons-from-the-batching-prototype)
  - [Removing generator overhead](#removing-generator-overhead)
  - [Specializing homogeneous operands](#specializing-homogeneous-operands)
  - [Speculative execution for mixed inputs](#speculative-execution-for-mixed-inputs)
  - [Extending the approach to N-ary expressions](#extending-the-approach-to-n-ary-expressions)
  - [Handling UNDEF and dates](#handling-undef-and-dates)
    - [Direct UNDEF where semantics permit it](#direct-undef-where-semantics-permit-it)
    - [Homogeneous dates and durations](#homogeneous-dates-and-durations)
  - [Classification and vectorization](#classification-and-vectorization)
- [Evaluation](#evaluation)
  - [Experimental setup](#experimental-setup)
  - [Three stages of evaluation](#three-stages-of-evaluation)
  - [Focused date and UNDEF controls](#focused-date-and-undef-controls)
    - [Date specialization](#date-specialization)
    - [Direct UNDEF handling](#direct-undef-handling)
- [Profiling findings](#profiling-findings)
  - [Historical flamegraphs](#historical-flamegraphs)
  - [Where execution time goes now](#where-execution-time-goes-now)
- [Discussion and conclusion](#discussion-and-conclusion)
  - [Limitations](#limitations)
  - [Remaining opportunities](#remaining-opportunities)
  - [Conclusion](#conclusion)

## Introduction and motivation

### Why optimize an addition?

Consider this SPARQL query:

```sparql
SELECT ?x ?y ?sum WHERE {
  VALUES (?x ?y) { (1 10) (2 20) (3 30) }
  BIND(?x + ?y AS ?sum)
}
```

`VALUES` supplies three rows with two variables, `?x` and `?y`. `BIND` evaluates
the expression `?x + ?y` for each row and assigns the result to `?sum`. The query
returns:

| `?x` | `?y` | `?sum` |
|---:|---:|---:|
| 1 | 10 | 11 |
| 2 | 20 | 22 |
| 3 | 30 | 33 |

For this example, QLever needs only three additions. In a larger query, the same
expression may be evaluated over millions of rows.

All input values in this example are integers. However, QLever's expression
evaluator must also handle other datatypes, such as doubles, and combinations of
different datatypes. The original evaluator determined how to interpret the inputs
separately for each row. Even when every row contained integers, it repeated this
work.

The goal of this project was to avoid repeated datatype handling where possible.
The optimized evaluator first checks the datatypes in the input columns. If a
column contains only integers, the evaluator can select an integer-specific
evaluation loop instead of repeating the general datatype handling for every row.

### The original evaluation path

QLever stores intermediate query results in tables. Each column corresponds to a
variable, such as `?x`, and each row contains one assignment of values to those
variables.

RDF identifies entities and predicates using Internationalized Resource
Identifiers (IRIs). An IRI is an identifier such as `https://example.org/person/1`.
RDF also uses literals to represent values such as numbers and text.

Each table entry occupies 64 bits and is represented in the code by the type
`Id`. An `Id` can represent an IRI or a literal. Some bits identify how to interpret
the remaining bits. We call these datatype bits the *tag*.

For integers and doubles, QLever stores a numeric representation directly in the
`Id`. Strings, including IRIs and text literals, are stored separately in a
dictionary called the *vocabulary*. Their `Id`s contain an index into that
dictionary. Numeric evaluation can therefore obtain integer and double values
without looking up a string in the vocabulary.

Now consider `?x + ?y` from the example. The addition has two operands: `?x` and
`?y`. Both refer to columns in the input table. An operand can also be a constant,
as in `?x + 10`, or another expression, as in `(?x * 2) + ?y`.

The original evaluator used *generators* to supply successive operand values. A
generator produces values as the evaluator requests them. For each row, the
addition evaluator performed these steps:

1. Obtain the next value from each operand's generator.
2. Interpret each value using a *value getter*. This is a function that converts
   an `Id` into a value accepted by the operation.
3. Select the addition operation for the resulting datatypes.
4. Add the values, encode the result as an `Id`, and append it to the output column.

A numeric getter could return an integer, a double, or a marker for a non-numeric
value. These alternatives were held in a C++ `variant`: an object that contains
one of several predefined types. The operation checked which alternatives the
getters had returned before performing the arithmetic. Date-aware addition also
supported date values.

This process handled different input datatypes through the same evaluation code.
However, it repeated generator traversal, value conversion, and datatype selection
for each row. Even when both input columns contained only integers, the evaluator
repeated those steps.

## Implementation

The implementation developed in several steps. The first prototype grouped
conversions into batches but retained much of the original work. The later
changes removed generator traversal and repeated datatype handling, then extended
those changes to more operations and datatypes.

### Lessons from the batching prototype

My first approach separated conversion from addition. Instead of converting two
`Id`s and adding their values immediately for each row, the prototype first
converted a batch of inputs into temporary vectors. Each vector contained
`NumericValue` objects. A `NumericValue` was a C++ variant holding an integer, a
double, or a marker for a non-numeric input. A second loop read the two temporary
vectors and computed the sums.

For the running example, the prototype would first convert the `Id`s in the `?x`
and `?y` columns into two vectors of numeric variants. The addition loop would
then read the corresponding variants, determine their types, and add their
values.

This approach grouped the conversions together, but it still converted every
input and checked the variants during addition. It also wrote the converted
values into temporary vectors and read them back. Early experiments showed that
the prototype could be slower than the original evaluator. Batching alone had
not removed the repeated datatype handling.

To investigate the cost, I used a standalone benchmark that constructed and
evaluated expressions directly, following the setup used by QLever's expression
unit tests. The benchmark did not run a complete query through the server. It
therefore let me examine expression evaluation separately from query planning
and result export. The historical flamegraphs in [Profiling findings](#profiling-findings)
show the conversion costs in these prototypes.

The lesson was to avoid an extra intermediate representation and reduce the work
needed to evaluate each row. The next change addressed how the evaluator accessed
its input values.

### Removing generator overhead

The next approach accessed input values by their row index instead of requesting
successive values from generators. This change introduced `BinaryExpression`,
the shared evaluator for operations with two operands, in
[PR #3205](https://github.com/ad-freiburg/qlever/pull/3205). Addition, subtraction,
multiplication, division, and power use this evaluator.

For `?x + ?y`, the evaluator obtains a view of each input column. A view refers to
the existing column data without copying it. For row `i`, the evaluator reads the
`Id`s at position `i` in both columns, applies the value getters and the addition
operation, and writes the result to the output vector. The loop no longer needs
to resume operand generators to obtain those values.

Constants need different handling. In `?x + 10`, the value `10` is the same for
every row. The evaluator can apply its value getter once and reuse the converted
value throughout the loop. If both operands are constants, as in `10 + 20`, the
evaluator returns a single value rather than constructing a result vector.

This change did not remove the datatype handling described earlier. For column
values, the getters still converted `Id`s into variants, and the operation still
checked those variants. It removed generator traversal and, for constant
operands, repeated conversion. The evaluation below compares this indexed
evaluator with both the original evaluator and the later datatype-specific
loops.

The new loop also had to preserve QLever's execution safeguards. Its output
vector uses QLever's allocator, which enforces the query's memory limit. The
loop checks for query cancellation at chunk boundaries, so a long evaluation
can still be interrupted. The same evaluation structure accepts different
operations and value getters; it is not a separate implementation for each
arithmetic operator.

### Specializing homogeneous operands

Indexed access removed generator traversal, but the evaluator still interpreted
the datatypes separately for each row. In the running example, both input columns
contain only integers. The next change used that information to select the
addition loop before processing the rows
([PR #3366](https://github.com/ad-freiburg/qlever/pull/3366)).

A column is *homogeneous* if every value in it has the same datatype. The evaluator
first examines the datatype tags of the `Id`s in each input column. We call this
step *classification*. If both columns contain only integers, the evaluator
selects a loop for two integer operands.

Inside that loop, the datatype of each input is already known. The evaluator
extracts the integer values directly from the `Id`s, adds them, and encodes the
result as an `Id`. It does not construct numeric variants or check which variant
alternatives are present for each row.

The two columns do not have to share a datatype. If `?x` contains only integers
and `?y` contains only doubles, the evaluator selects the loop for an integer
and a double. Each column must be homogeneous, but their datatypes can differ.
For a constant operand, such as `10` in `?x + 10`, the evaluator checks its
datatype and extracts its value once.

The following pseudocode summarizes this choice:

```text
examine the datatype tags in each input column
if each operand has one supported datatype:
    select the loop for that combination of datatypes
    for each row:
        extract values of the known datatypes
        compute the operation and encode its result as an Id
else:
    try the mixed-input optimization described next
    or use the general evaluator
```

The specialized loops preserve the operation's existing arithmetic and result
encoding. They still allocate and fill an output vector. They also preserve the
operation's rules for invalid results. For example, if an operation maps a
non-finite result, such as infinity, to an undefined value, its specialized loop
must do the same. We use *UNDEF* to refer to an undefined value in the rest of
this post.

Classification itself takes time because it examines the input columns before
the arithmetic loop. The complete-expression measurements below include that
work, together with result allocation and construction. Measuring only the
specialized arithmetic loop would overstate the improvement.

### Speculative execution for mixed inputs

The homogeneous optimization requires every value in a column to have the same
datatype. Consider a column with 99 integers and one double. It is no longer
homogeneous, so the homogeneous optimization cannot be used. Yet most rows could
still use integer arithmetic.

[PR #3417](https://github.com/ad-freiburg/qlever/pull/3417) added an optimization
for such mixed columns. During classification, the evaluator counts the values
of each datatype. It selects a *preferred datatype* if one datatype occurs more
often than every other datatype and is supported by the specialized evaluator.

Suppose both `?x` and `?y` contain mostly integers. The evaluator selects integer
arithmetic as the expected case. For each row, it checks the tags of both input
`Id`s before extracting their values:

- If both inputs are integers, it extracts the integers directly and computes
  their sum.
- If either input has another datatype, it sends that row to a separate function
  that handles the exceptional inputs.

We call this *speculative execution*: the evaluator chooses a likely datatype
combination, but checks that the current inputs match before using it. It never
interprets a double or a string reference as an integer merely because most
other rows contain integers.

For binary expressions, the exceptional-input function handles other integer
and double combinations directly. For remaining cases, it uses the existing
value getters and general operation. For example, boolean inputs still use the
numeric getter's existing conversion to integers. Keeping this handling in a
separate function keeps the loop for matching rows small.

A preferred datatype need not account for more than half the column. For
example, a column with 40 integers, 35 doubles, and 25 undefined values has
integer as its unique most frequent datatype. A column with 50 integers and
50 doubles has a tie, so neither datatype is preferred. The counts include all
datatypes, not just the numeric ones. A column dominated by strings must not be
treated as predominantly integer simply because integers outnumber doubles.

The speculative loop is used only when every operand has a supported preferred
datatype. Otherwise, evaluation uses the general loop. Mixed columns can
therefore benefit less than homogeneous columns: matching rows still need tag
checks, exceptional rows need additional handling, and ties can prevent this
optimization entirely. The evaluation below includes both mostly integer columns
and evenly mixed columns to show that difference.

### Extending the approach to N-ary expressions

Addition has two operands and is therefore a *binary* operation. Other
expressions take a different number of operands. For example, `ABS(?x)` takes
one operand and returns its absolute value. An operation with N operands is
called *N-ary*. QLever's `NaryExpression` evaluator supports operations with
different numbers of operands, including one.

[PR #3604](https://github.com/ad-freiburg/qlever/pull/3604) extended indexed
evaluation to this shared evaluator. As with binary expressions, each operand
can supply a column, a vector computed by another expression, or a constant.
For each row, the evaluator obtains the corresponding operand values by index,
applies their value getters, and calls the operation. This replaces generator
traversal in the general evaluation loop. Expressions that already have their
own specialized evaluation remain handled separately.

Indexed access and numeric specialization are two distinct improvements. Indexed
access can be used for non-numeric operations as well. For example, expressions
that process strings use the indexed loop while retaining their existing value
getters and operations. They do not interpret string references as numbers.

Numeric expressions can additionally use the homogeneous and speculative
optimizations described above. For `ABS(?x)`, an all-integer column allows the
evaluator to select the integer loop before processing the rows. A mostly
integer column can use the speculative loop, with checks for exceptional values.
In production, the numeric specializations added through `NaryExpression` apply
to unary numeric operations, including unary minus, `ABS`, `ROUND`, and `LOG`.
String, language, geographic, and conditional expressions receive the indexed
loop improvement rather than numeric specialization.

Sharing these loops also requires controlling compilation cost. Each additional
operand increases the number of possible datatype combinations. Generating a
specialized C++ loop for every combination would increase the work required from
the compiler. The implementation therefore checks which combinations an
operation supports before generating those loops. For example, date combinations
are generated only for operations whose getters support dates. Other cases
retain the general evaluator.

The shared implementation also had to compile correctly with QLever's supported
compilers. Some expressions needed named temporary variables to avoid
code-generation failures in Clang 16 and 17.

To test the shared numeric machinery with more than two operands, the evaluation
below includes a three-argument sum constructed for the benchmark. That sum is
a test operation, not an additional SPARQL built-in function. Its results show
how the machinery behaves for that example; they do not measure every expression
that uses `NaryExpression`.

### Handling UNDEF and dates

The previous optimizations make numeric evaluation faster when the inputs have
suitable datatypes. Two further changes address inputs outside the integer and
double cases. The first avoids unnecessary work when an input already guarantees
an undefined result. The second extends specialized evaluation to dates and
durations.

#### Direct UNDEF where semantics permit it

Consider multiplication, `?x * ?y`, with an integer in `?x` and an IRI in `?y`.
The numeric getter does not accept the IRI as a number. The operation therefore
cannot produce a numeric result, regardless of the integer's value. The evaluator
represents this undefined result as UNDEF.

The general evaluation path discovers this through the value getters and the
operation. [PR #3708](https://github.com/ad-freiburg/qlever/pull/3708) allows the
evaluator to identify such cases from the datatype tags and write UNDEF directly.
It can skip constructing numeric variants and calling the arithmetic operation
for that row.

This shortcut is enabled only for operations that explicitly guarantee that an
input rejected by their getter makes the result undefined. The rejection check
must follow that getter's rules. For example, QLever's numeric getter accepts
booleans and converts them to integers. The shortcut must therefore not reject a
boolean simply because its tag differs from the integer and double tags.

Dates need a different check. A date is not an integer or a double, but it can
participate in valid date arithmetic. The initial change left date-aware addition
and subtraction on their existing general path. It enabled direct UNDEF handling
for eligible unary numeric operations, multiplication, division, and power. The
subsequent date change extended the rejection check to the date-aware operations
while preserving accepted date inputs.

Classification also records whether a column contains any datatype that the
getter rejects. If every input datatype is accepted, the evaluator can omit the
rejection checks inside the loop. Otherwise, those checks save work on rejected
rows but also cost time on accepted rows. The evaluation therefore tests both
fully numeric columns and columns containing undefined inputs.

#### Homogeneous dates and durations

Date subtraction illustrates why the same optimization is useful beyond ordinary
numbers. Suppose two columns contain dates and an expression subtracts one date
from the other. The result is a duration: a length of time. If the evaluator knows
that both columns contain date values, it can avoid repeatedly selecting the
date-related alternatives from variants for every row.

[PR #3737](https://github.com/ad-freiburg/qlever/pull/3737) extended the existing
classification and specialized loops to these inputs. QLever uses a C++ type
called `DateYearOrDuration` to represent its supported date, year, and duration
values after extraction from an `Id`. For operations with date-aware getters, the
evaluator can select a loop that extracts this representation directly and calls
the existing date arithmetic.

Recognizing a date-related representation does not establish that every operation
on it is valid. A date and a duration, for example, have different meanings even
though they use the same C++ representation. The existing arithmetic code still
checks which combinations are supported and applies their date, timezone, and
duration rules. The optimization changes how values reach that code; it does not
replace date arithmetic with subtraction of the ``Id``s' raw bits.

Mixed columns can use the same speculative approach as numeric columns. Rows
matching the selected date-related datatype use the specialized operation.
Exceptional rows retain the checks and general handling needed for their inputs.
The date optimization therefore shares the evaluation machinery rather than
introducing a separate evaluator with different arithmetic rules.

### Classification and vectorization

Classification determines whether a column is homogeneous and counts the values
of each datatype for speculative evaluation. These counts form a *histogram*:
one counter per datatype. A straightforward implementation would read each `Id`,
extract its tag, and increment the corresponding counter. The final classifier
instead processes blocks of 32 ``Id``s. This organization lets it count common
patterns with fewer histogram updates.

For each block, the classifier first checks whether all tags match the first
tag. It computes the bitwise exclusive OR (XOR) of each tag with the first tag.
XOR produces zero for equal tags and a nonzero value for different tags. The
classifier then combines these differences using bitwise OR. The combined result
is zero exactly when every tag matches. In that case, the classifier adds 32 to
the counter for that datatype.

For a mixed block, the classifier tries counting occurrences of two candidate
datatypes. It checks that those counts account for all 32 entries. If they do,
it adds the two counts to the histogram. If the block contains additional
datatypes, it uses the general loop that updates a counter for each entry.
Any entries remaining after the last complete block are handled separately.

This block-based code is more involved than a loop that stops at the first
mismatching tag. However, an early-exit version tested during development made
homogeneous input slower. The regular block checks also give the compiler an
opportunity to process multiple tags with one instruction.

Processing several values with one instruction is called *single instruction,
multiple data* (SIMD). When a compiler transforms an ordinary loop into SIMD
instructions, the transformation is called *autovectorization*. The block checks
use ordinary C++ code rather than explicitly written SIMD instructions.

For the GCC 13.3 build examined here, the compiler's optimization reports indicate
16-byte vectorization for tag extraction, the XOR/OR check, and the two-datatype
counting loops. The general loop that updates the histogram for each entry was
not vectorized. Inspection of the generated machine instructions also found SIMD
operations in the classifier and in the specialized arithmetic loops.

These observations establish which loops this compiler vectorized. They do not
isolate how much of the overall speedup comes from vectorization: the evaluator
also removes generators and repeated variant handling. Nor do they establish
whether explicitly written SIMD would be faster. That would require a separate
implementation and a controlled comparison.

For an expression with a fixed number of operands, classification takes time
proportional to the number of input rows: it examines each input `Id` once.
Evaluation also processes each row once. If there are n rows, both passes
therefore take O(n) time, assuming constant-time operations such as numeric
addition. The original evaluator was linear too; the optimization reduces the
work per row rather than changing that growth rate. The classifier uses a fixed
set of datatype counters. A vector result needs O(n) output space, while an
expression with only constant operands can return a single value.

## Evaluation

The evaluation asks how much faster the changed evaluator is and which inputs
benefit. The first experiment compares three source snapshots from the project.
The second enables and disables the date and UNDEF optimizations within the same
source version to examine those changes more directly.

### Experimental setup

The first experiment compares three versions of QLever. Each version is a
snapshot of the source code at a particular Git commit. The commit identifiers
below specify exactly which source code was built; the stage names explain
why those versions were selected.

| Stage | Evaluation approach | Git commit |
|---|---|---|
| Original evaluator | Before PR #3205; uses generators and general datatype handling | `6358e102802b8c192630814fb8340d2e8dee2c49` |
| Indexed evaluator | Before PR #3366; uses indexed binary evaluation, without the homogeneous numeric optimization | `9290ebe087ac7a8c20253fe4b8ca40f7ca893949` |
| Final project version | Includes the later numeric optimizations and date handling | `f86881abd8758132017a2d9a747b0fd645dcd79d` |

The middle stage helps distinguish the improvement from indexed binary access
from the subsequent improvement with numeric specialization. However, these
snapshots also include unrelated changes: 216 commits separate the original and
final versions. The experiment therefore compares complete source snapshots. It
does not measure the contribution of each PR in isolation.

All three versions were compiled with GCC 13.3 in C++20 mode, using the same
Release optimization settings (`-O3 -DNDEBUG`). Link-time optimization, which
allows optimization across separately compiled files, was disabled. No
`-march=native` option was added to specialize the build for the host processor.
The versions used the same copied set of dependency sources, rather than each
snapshot's original dependency versions. Separate source and build directories
kept the experiment independent of the development checkout.

Each test case specifies an expression and a distribution of input datatypes.
The input table contains 100,000 rows. One timed batch evaluates the expression
50 times over that table. A time in the results table therefore describes the
whole batch, not one addition or one query.

Each test case was run in ten independent processes for each source snapshot.
Before timing, each process performed three warmup evaluations and checked every
output row against its expected result. Input data was generated before timing.
For mixed columns, a fixed random seed made the shuffled datatype distribution
reproducible.

The timed region includes evaluating the operands, classifying their datatypes,
allocating the output, computing and storing results, and releasing the results.
A compiler memory barrier prevents the compiler from discarding the result
computations. The order of source versions was varied between runs to reduce
systematic effects from running one version first.

The measurements ran on a Ryzen 7 3700X. Each benchmark process was restricted
to logical CPU 2 to prevent migration between CPUs. The machine was shared with
other work. Its power policy and automatic frequency boost were left unchanged,
and the other hardware thread sharing that physical core was not reserved.
These conditions can cause timing variation, even with a fixed CPU assignment.

### Three stages of evaluation

The table compares the three source snapshots using addition and the
three-argument test operation introduced earlier. The input labels describe
where the operand values come from:

- **Vector-vector:** both operands supply one value per row, as in `?x + ?y`.
- **Vector-constant:** one operand supplies a value per row and the other is a
  fixed value, as in `?x + 10`.
- **Mixed inputs:** the percentages describe the datatype distribution in the
  input columns. The exceptional datatypes are shuffled rather than grouped
  together.
- **Ternary:** the benchmark's three-argument sum computes left + right + left.
  It reads the left input twice and is not a SPARQL built-in function.

Each time is the median of ten measured batches, in milliseconds. Each batch
contains 50 evaluations over 100,000 rows. A smaller time is better. For each
sample pair, the original time is divided by the final time to obtain a speedup;
the last column reports the median of those ten ratios. It is not necessarily
the ratio of the two displayed medians. A speedup of 2× means that the final
version takes half as long in that comparison.

| Input case | Original (ms) | Indexed (ms) | Final (ms) | Original-to-final speedup |
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

For integer vector-vector addition, the median batch time falls from 109.45 ms
in the original evaluator to 73.11 ms in the indexed evaluator. The indexed
stage still uses general datatype handling. The final version, which can select
an integer-specific loop, takes 6.64 ms. Integer vector-constant addition follows
the same pattern, with times of 97.30, 47.20, and 3.85 ms. These results are
consistent with removing generator traversal first and repeated datatype
handling afterward. Because other code also changed between snapshots, they do
not isolate the contribution of each PR.

The ternary results help explain the scope of the middle stage. That stage
changed binary evaluation, so it did not introduce the indexed N-ary evaluator.
Its ternary times remain close to those of the original version. The final
version includes the N-ary changes and numeric specialization; its homogeneous
integer case takes 9.44 ms rather than 147.74 ms.

Mixed columns benefit less. The binary 90% integer / 10% double case has an
overall speedup of 7.53×, compared with 16.47× for all-integer vector-vector
input. Matching rows can use speculative integer evaluation, but exceptional
rows require separate handling.

An equal mixture exposes a regression relative to the middle stage. For 50%
integer / 50% double input, the final version takes 78.85 ms, compared with
74.50 ms for the indexed evaluator: about 5.8% longer. Neither datatype is uniquely
most frequent, so the evaluator cannot select a preferred datatype. It performs
classification before using general evaluation. The final version still beats
the original version, but that comparison alone would hide the regression.

The experiment contains 300 timed batches across ten input cases and three
source snapshots. The raw measurements retain the ranges, including occasional
slow runs on the shared machine. The results concern these expressions, input
sizes, and datatype distributions. In particular, the ternary operation reuses
an operand, and its result should not be generalized to arbitrary N-ary
expressions. Nor should the large homogeneous gains be treated as gains for
complete SPARQL queries.

### Focused date and UNDEF controls

The source snapshots above contain many changes. To examine the date and
UNDEF optimizations more directly, a second experiment compares two variants
within the final source version. One variant enables the optimization under
study; the other disables it. These switches are implemented in types used only
by the benchmark. They do not change QLever's production evaluator.

Each variant uses 100,000 rows and 50 evaluations per timed batch. There are ten
measured batches per variant and 260 measurements in total. Each process checks
all output rows before timing. The compiler and CPU assignment are the same as
in the three-stage experiment. Times are medians in milliseconds, and speedups
are medians of paired disabled/enabled timing ratios.

These measurements start after the operands have been evaluated. They include
classification, output allocation, computation, and result destruction, but
exclude the work needed to evaluate the operand expressions. Their absolute
times therefore should not be compared directly with the complete-expression
times above. They also do not recreate historical versions of the PRs.

#### Date specialization

The date experiment uses subtraction. With date specialization disabled, date
inputs pass through the general getter and operation. With it enabled, eligible
date inputs use the specialized loop. Both variants preserve the same date
conversion and subtraction rules. Numeric specialization remains available in
both variants, and direct UNDEF handling is disabled in both. This keeps the
comparison focused on date specialization.

The test data repeats small sets of date and duration values. Expected valid
results are constructed independently as one-day durations. The dateTime case
also includes different timezone offsets. Integer and double subtraction serve
as comparison cases: enabling date specialization should not materially change
their execution.

| Input case | Disabled (ms) | Enabled (ms) | Speedup |
|---|---:|---:|---:|
| Date subtraction, vector-vector | 253.867 | 182.991 | 1.38× |
| Date subtraction, vector-constant | 224.315 | 182.758 | 1.23× |
| dateTime subtraction, vector-vector | 255.486 | 185.694 | 1.38× |
| Duration subtraction, vector-vector | 208.117 | 105.392 | 1.98× |
| Date subtraction, 30% UNDEF | 201.787 | 153.942 | 1.31× |
| Integer subtraction control | 6.618 | 6.629 | 1.00× |
| Double subtraction control | 4.866 | 4.869 | 1.00× |

For vector-vector input, date and dateTime subtraction are about 1.38× faster
with specialization enabled. Duration subtraction is about 1.98× faster. The
integer and double comparison cases are nearly unchanged. These observations
apply to the tested subtraction cases; they do not establish the same gains
for every date operation or input distribution.

The [merged date/duration PR](https://github.com/ad-freiburg/qlever/pull/3737)
also reports two measurements on larger inputs from Wikidata:

| Reported workload | Input size | Before | After |
|---|---:|---:|---:|
| `?death - ?birth` | 3.7 million humans with both dates | 0.39 s | 0.34 s |
| `?birth + "P30D"^^xsd:dayTimeDuration` | 8.1 million dates | 0.31 s | 0.26 s |

The first expression subtracts birth dates from death dates. The second adds a
30-day duration to birth dates. These measurements are taken from the merged PR
description, not repeated in the experiments reported here. The description
does not specify the full environment or exactly which work was timed, so these
times should not be compared directly with the benchmark times above. The PR
also reports regressions for some mixtures, reinforcing the need to test inputs
beyond homogeneous columns.

#### Direct UNDEF handling

The UNDEF experiment uses multiplication. Both variants use the same numeric
getters and arithmetic. The enabled variant allows the evaluator to write UNDEF
directly when an input's datatype guarantees that result. The disabled variant
uses the existing getter and operation to obtain it.

Undefined entries are shuffled among valid entries. The 35% integer / 35% double
case has no unique most frequent datatype, so it cannot select a preferred
numeric datatype for speculative evaluation. The boolean case checks inputs
that the numeric getter accepts by converting them to integers. The all-integer
and all-double cases show the effect when there are no undefined inputs to skip.

| Input case | Disabled (ms) | Enabled (ms) | Speedup |
|---|---:|---:|---:|
| 70% integer / 30% UNDEF | 33.767 | 24.119 | 1.40× |
| 70% double / 30% UNDEF | 33.477 | 24.173 | 1.39× |
| 35% integer / 35% double / 30% UNDEF | 81.371 | 64.346 | 1.27× |
| 70% boolean / 30% UNDEF | 78.563 | 59.713 | 1.32× |
| All-integer control | 8.435 | 8.404 | 1.00× |
| All-double control | 6.656 | 6.791 | 0.98× |

Direct UNDEF handling reduces the median batch time for all four tested
mixtures. For example, the 70% integer / 30% UNDEF case falls from 33.77 to
24.12 ms, a paired speedup of about 1.40×. The tied integer/double mixture also
benefits, even though it cannot use a preferred numeric datatype.

The fully numeric cases do not show a uniform improvement. The integer case is
nearly unchanged, while the all-double median is about 2% slower with direct
UNDEF handling enabled. Small differences can depend on compiler output and
variation on the shared machine. These results show a benefit for the tested
undefined-input mixtures, not a guarantee that enabling the shortcut is free
for all inputs.

## Profiling findings

Timing measurements show how long an evaluation takes. Profiles help identify
which parts of the evaluator account for that work. This section first discusses
the historical flamegraphs that guided development, then compares profiles of
the original and final evaluators.

### Historical flamegraphs

During development, I collected profiles to investigate which functions accounted
for most of the sampled execution. The profiles were recorded with
`perf record --call-graph fp`. This command samples a running process and records
its call stacks: the chain of functions active at each sample. I then processed
the recording with `perf script` and the FlameGraph scripts to produce interactive
SVG images.

A flamegraph groups samples that contain the same call stacks. Each rectangle
represents a function in a stack. Rectangles above it represent functions called
from that stack. A wider rectangle means that more samples contain that frame;
the horizontal position does not represent when the function ran.

The percentages below are *inclusive*: a function's share includes samples in
functions it calls. For example, if conversion is called inside addition, the
same sample can contribute to both rectangles. Their percentages must not be
added as if they were separate costs.

| Prototype profile | Important inclusive profile weights | Finding |
|---|---|---|
| Initial batching prototype (July 2) | Addition evaluator 94.82%; nested `getNumericBatch` 75.49% | Investigate conversion/materialization before the arithmetic. |
| Integer-specialized conversion (July 7) | Conversion subtree 47.50%; separate variant frame 27.29%; `isAllInt` 5.36% | Specializing conversion still leaves intermediate representation costs. |
| DBLP query (July 14) | Addition evaluator 52.41%; nested multiplication 51.42%; nested YEAR evaluator 20.18% | Much of the apparent addition cost belongs to child expressions. |

In the first prototype, `getNumericBatch` converted ``Id``s into temporary numeric
vectors. Its large share pointed to conversion and vector construction as costs
to investigate. Specializing integer conversion still left conversion and variant
handling prominent. In that graph, `getNumericBatch` and its nested function
`getNumericBatchInt` both show 47.50% because they cover the same samples.

The DBLP profile illustrates the importance of inclusive shares. Most samples
under addition also occur in its multiplication operands, which include extracting
years from dates. The addition rectangle therefore does not isolate addition's
own work. Likewise, a result-export frame can include query computation performed
while results are requested, rather than serialization alone.

Open the full graphs in new tabs to inspect those call stacks. Click a rectangle
to zoom and use the search control to find a function:

- <a href="img/flamegraph-batched-add-2026-07-02.svg" target="_blank" rel="noopener">Batch conversion dominates the first prototype</a>
- <a href="img/flamegraph-integer-conversion-2026-07-07.svg" target="_blank" rel="noopener">Integer conversion still leaves variant overhead</a>
- <a href="img/flamegraph-dblp-direct-integer-2026-07-14.svg" target="_blank" rel="noopener">DBLP: child expressions dominate the addition frame</a>

These graphs document prototype development, not a controlled comparison of the
final implementation. Percentages are relative to each recording, so a larger
share does not establish a larger absolute cost. The SVGs do not specify exact
source snapshots or sampled event details, and the raw recordings are not
included. The `fp` mode also requires frame pointers in the compiled code to
reconstruct stacks reliably; that build setting has not been verified here.
An <a href="img/flamegraph-initial-unsymbolized-2026-06-25.svg" target="_blank" rel="noopener">initial server profile with unresolved symbols</a>
contains many frames without function names and offers less useful attribution.

### Where execution time goes now

A separate set of benchmark profiles compares the original and final evaluators
using GNU user-space CPU sampling. At 100 Hz, the profiler samples the current
instruction address about 100 times per second of sampled CPU time. Each profile
contains approximately 1,000–1,200 samples in the executable's machine code.

This is a *flat* profile: each sample is assigned to the code region executing
at that moment, without reconstructing its callers. Unlike flamegraph shares,
these percentages do not include all called functions. They exclude shared-library
code, including some memory-allocation and C library work.

The original generators use coroutines: functions that can suspend and resume.
The default `gprof` report assigned some coroutine code to neighboring functions.
I instead matched the recorded address ranges to function boundaries in the
compiled executable and left gaps unresolved. Four-byte recording ranges make
attribution near boundaries approximate. Profiling and elapsed-time measurements
were collected in separate runs.

The table shows the main sampled regions before and after optimization:

| Workload | Original getter | Original operation coroutine | Final classification | Final specialized loop |
|---|---:|---:|---:|---:|
| Integer vector-vector | 60.7% | 16.1% | 33.9% | 65.4% |
| Double vector-vector | 59.5% | 16.1% | 41.5% | 57.9% |
| Integer vector-constant | 46.7% | 20.7% | 27.9% | 71.9% |
| Ternary integer (left + right + left) | 65.6% | 14.0% | 31.2% | 68.4% |

For integer vector-vector addition, the original getter and operation coroutine
account for most samples. In the final evaluator, classification and the
specialized loop dominate instead. That loop includes reading ``Id``s, extracting
numbers, arithmetic, encoding, and output stores; its share is not the cost of
the addition instruction alone. The double and ternary cases show the same shift.

Each profile has its own total. A larger percentage in the faster evaluator need
not mean a larger absolute cost, and these shares cannot be used as an exact
decomposition of the separate elapsed-time measurements.

A separate experiment timed only classification of two integer columns. It took
about 1.93 ms per batch. That experiment checked numeric datatypes only, whereas
production addition also recognizes dates. It confirms that classification has
a measurable cost, but it is not an exact measurement of the classification
component inside the complete addition benchmark.

The final evaluator behaves differently on evenly mixed numeric input. For the
50% integer / 50% double case, about 61% of samples still fall in the general
getter and about 4.4% in classification. Neither datatype is preferred, so the
specialized speculative loop is unavailable. This agrees with the timing results:
the homogeneous optimization removes repeated datatype handling for uniform
columns, but does not remove it for every mixed column.

## Discussion and conclusion

The results apply to the expressions and inputs tested here. This section
explains the limits of those measurements, discusses possible improvements
suggested by the profiles, and summarizes what the project achieved.

### Limitations

These expression benchmarks do not measure complete query execution. Finding
data, combining rows, and producing output may dominate a query's time and limit
the benefit from faster expressions. The historical DBLP profiles and the
Wikidata measurements reported in the merged date PR provide larger-input
context, but were not repeated in the local experiments.

Numeric inputs repeat values modulo 1,000 to avoid overflow in the tested
arithmetic. Date inputs likewise repeat small sets of values. Checking every
output establishes correctness for those inputs, not all arithmetic edge cases.
Existing numeric and date helper tests also passed, including classification
block boundaries and exceptional inputs. The full SPARQL test suite was not rerun
as part of this evaluation campaign.

The results cover one compiler, one processor, and one input size on a shared
machine. They do not establish how gains change with compiler output, input sizes
that interact differently with processor caches, or mixed datatypes arranged in
groups rather than shuffled. Those conditions require additional measurements.

### Remaining opportunities

The profiles suggest different follow-ups for homogeneous and mixed inputs:

- **Reuse classification for repeated operands.** The ternary benchmark reads
  the left column twice. Reusing its classification could avoid a second scan.
  The evaluator must establish that both operands share data and require the
  same checks. Measurements must include that detection cost, even when no
  operands can share a classification.
- **Specialize evenly mixed numeric inputs.** A loop could handle integer/double
  combinations without a preferred datatype or general variants. It would still
  need datatype checks and the existing arithmetic rules. Comparing it on both
  mixed and homogeneous inputs would reveal whether those checks introduce
  regressions.
- **Investigate the specialized loop.** Its large profile share includes loads,
  numeric extraction, arithmetic, encoding, and stores. Some loops already use
  compiler-generated SIMD. Explicit SIMD would need a controlled comparison;
  profile width alone does not establish that it would help.

These are unimplemented experiments, not claimed speedups. Any change must also
preserve memory limits, query cancellation, and correctness.

### Conclusion

This project reduced the work surrounding arithmetic in QLever's expression
evaluator. The original evaluator obtained values through generators, converted
them with value getters, and selected operations from variants for each row.
The changes first replaced generator traversal with indexed access. They then
used column classification to select specialized loops for homogeneous inputs
and checked speculative loops for mostly uniform inputs. Further changes avoided
unnecessary work for undefined results and extended specialization to dates and
durations.

The experiments show the largest gains on homogeneous input. Integer
vector-vector addition improved from 109.45 to 6.64 ms per measured batch across
the original and final source snapshots. Mixed inputs benefited less. An even
integer/double mixture remained faster than the original evaluator, but was
about 5.8% slower than the intermediate indexed evaluator. The focused date and
UNDEF experiments also showed benefits for their tested cases, alongside small
regressions in some comparison cases. These are expression-level results, not
speedups for complete queries.

Profiling explains the difference between the input cases. The original profiles
place much of the sampled execution in getters and coroutine code. For the final
homogeneous evaluator, classification and the specialized loop account for most
samples. Evenly mixed numeric input still spends much of its sampled execution
in the general getter. The remaining costs therefore depend on the input's
datatype distribution. Further optimization should target those measured costs
and check both correctness and performance across different input cases.
