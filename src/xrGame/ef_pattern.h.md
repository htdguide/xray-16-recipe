# src/xrGame/ef_pattern.h

> Declares the trained, table-driven evaluation function implemented in [`ef_pattern.cpp`](ef_pattern.cpp.md), and the indexing scheme its tables are addressed by.

**Needs** — [`ef_base.h`](ef_base.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`ef_pattern.cpp`](ef_pattern.cpp.md) · [`ef_storage.cpp`](ef_storage.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md)
**Tier floor** — T3: a declaration plus one index computation

## Purpose

Declares `CPatternFunction`, an [evaluation function](ef_base.h.md) whose answer is not
computed but *looked up* in a table fitted offline by a separate tool. Substance in
[`ef_pattern.cpp`](ef_pattern.cpp.md); the index computation is here and is substance too.

## State

```text
RECORD PatternFunction EXTENDS BaseFunction
  header           : { builder_version : int, data_format : int }
  variable_count   : int
  variable_types   : list<int>      # slot number of the input function for each variable
  feature_range    : list<int>      # how many buckets each variable is discretized into
  variable_values  : list<int>      # scratch: this evaluation's bucket per variable
  function_type    : int            # the catalogue slot THIS function claims
  patterns         : list<Pattern>
  pattern_offsets  : list<int>      # where each pattern's block starts in `parameters`
  parameters       : list<real>     # the fitted weights, one flat array

RECORD Pattern
  variable_indexes : list<int>      # which variables this pattern is over
```

**Invariant** — a *pattern* is a subset of the input variables, and the model's answer is the
**sum of one fitted weight per pattern**. So a pattern over two variables contributes a
weight chosen by the pair; a pattern over one contributes a weight chosen by that variable
alone. This is an additive model over feature conjunctions — the structure a maximum-entropy
or log-linear classifier produces — and the training tool that fitted it is not in this
repository.

**Invariant** — `parameters` is one flat array holding every pattern's block end to end, and
`pattern_offsets` says where each block starts. A pattern's block is as large as the product
of its variables' bucket counts.

## `dwfGetPatternIndex`

**Contract** — the mixed-radix index of one pattern's weight, given the current bucket of
every variable. Pure.

```text
FUNCTION pattern_index(values, p) -> int
  pat = patterns[p]
  index = values[pat.variable_indexes[0]]
  FOR EACH subsequent variable v IN pat.variable_indexes
    index = index * feature_range[v] + values[v]     # mixed radix, most significant first
  RETURN index + pattern_offsets[p]
```

**Invariants** — the radix of each digit is that *variable's* bucket count, read from the
global feature-range list rather than from anything pattern-local. The digits are ordered
most-significant first, matching the order the pattern's variables are listed in. Both facts
are frozen by the shipped tables: any other ordering reads a different weight.

**Notes** — the radix is looked up by the variable's index within the pattern for the first
digit but by its global index for the rest — a subtle inconsistency in the original that is
harmless only because the first digit needs no radix.

## Exported units

- construction from a data-file name, and destruction.
- `vfLoadEF` — parse one trained-function file and register into the catalogue.
- `ffGetValue` — discretize every input, sum the pattern weights, optionally trace.
- `ffEvaluate` — the sum itself.
