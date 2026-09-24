# src/xrGame/ef_pattern.cpp

> Loads a trained evaluation function from its data file and answers by summing one fitted weight per feature pattern.

**Needs** — [`ef_pattern.h`](ef_pattern.h.md) · [`ef_primary.h`](ef_primary.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ai_space.h`](ai_space.h.md) · [`ai_debug.h`](ai_debug.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — reached through its declarations in [`ef_pattern.h`](ef_pattern.h.md); callers name that, not this file.
**Tier floor** — T1: reads a binary file whose layout is frozen by shipped data

## Purpose

The alife simulation makes decisions about creatures nobody is looking at — whether they win
a fight, whether they breed, whether they flee an anomaly — and it makes them by evaluating a
model that was **fitted offline** by a tool that does not ship with this engine and whose
source is not in this repository. This file is the inference half of that: it reads the
fitted weights and evaluates them.

That means the model cannot be retrained from anything here. The data files are as frozen as
any other shipped asset, and the twenty-four of them are a substantial part of the game's
behaviour that a rebuild must load rather than reimplement.

## State

See [`ef_pattern.h`](ef_pattern.h.md).

## The data-file format

**Frozen.** Read strictly in order, little-endian, no padding:

```text
header
  builder_version : int (32-bit)   # must equal 1; any other value is a hard failure
  data_format     : int (32-bit)   # read and never used
variable_count    : int (32-bit)
feature_range     : int (32-bit) × variable_count   # buckets per input variable
variable_types    : int (32-bit) × variable_count   # catalogue slot of each input function
function_type     : int (32-bit)                    # the slot THIS function claims
min_result        : real (32-bit)
max_result        : real (32-bit)
pattern_count     : int (32-bit)
FOR EACH pattern
  cardinality     : int (32-bit)
  variable_indexes: int (32-bit) × cardinality      # indexes into variable_types
parameters        : real (32-bit) × (sum over patterns of the product of their
                                     variables' feature ranges)
```

**Invariants** — the parameter count is **not stored**; it is derived by walking the patterns
and multiplying their variables' bucket counts. A file whose weight block does not match that
derivation is read past its end or short, with no diagnostic. This is the format's one real
hazard and a rebuild should store the count.

**Invariants** — the version check is exact equality against one, and a mismatch is a hard
failure with a message naming the training tool. There is no migration path.

**Invariants** — the declared minimum and maximum come from the file, not from code, because
they are properties of the fit.

**Notes** — the loader computes a running prefix sum of the feature ranges into a local array
and then discards it unused. It is the remains of an earlier flat-indexing scheme.

**Notes** — the whole file is read with unaligned struct reads from a mapped region; see
[Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) and the platform
assumptions in the system requirements. A rebuild may parse field by field.

## `vfLoadEF`

**Contract** — resolves the file under the AI data root, reads it as above, allocates the
scratch value array, and — the load-bearing side effect — **registers this function into the
catalogue at the slot its own `function_type` field names**. Finally it takes the function's
name from the file's base name, which is how scripts address it.

**Invariants** — the function's catalogue slot comes from the data, not from the code that
constructed it. So the twenty-four construction calls in
[`ef_storage.cpp`](ef_storage.cpp.md) do not determine where their functions land; the files
do. Two files claiming the same slot silently overwrite, and a file claiming a primary
function's slot silently replaces it.

**Invariants** — a missing file is a hard failure. The trained functions are not optional.

## `ffGetValue`

**Contract** — evaluates the model against whatever is currently in the shared parameter
block. Not pure: it drives every input function, each of which reads the block.

```text
FUNCTION value() -> real
  FOR EACH variable i
    values[i] = catalogue[variable_types[i]].discrete(feature_range[i])
  RETURN sum over patterns p of parameters[pattern_index(values, p)]
```

**Invariants** — each input is discretized into exactly the number of buckets *the trained
data was fitted with*, read from the file. This is where the bucket count in
[`ef_base.h`](ef_base.h.md) comes from, and it is why discretization cannot be a property of
the function being discretized.

**Invariants** — evaluation is a nested call into other evaluation functions, which read the
same shared parameter block. It is safe only because no input function modifies the block —
except through the enemy-perspective wrapper, which saves and restores it. See
[`ef_storage.h`](ef_storage.h.md).

**Notes** — in a debug build with the AI-function trace flag set, the evaluation is run,
formatted into a line naming the function and every bucket index (printed one-based), logged,
and returned. It is the only visibility into why a trained function answered what it did, and
it is worth keeping in a rebuild — a fitted table is otherwise opaque.

## `ffEvaluate`

**Contract** — the sum itself, over the current bucket values. Pure with respect to the
parameter block.

## Destruction

**Contract** — frees every allocated array, including each pattern's own variable-index list.
