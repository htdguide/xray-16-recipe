# src/xrGame/ef_base.h

> What every evaluation function must provide: a real-valued answer, a declared range, and the rule that turns the answer into one of a few discrete buckets.

**Needs** — [`ef_storage.h`](ef_storage.h.md)
**Used by** — [`ef_pattern.h`](ef_pattern.h.md) · [`ef_primary.h`](ef_primary.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`ef_storage_script.cpp`](ef_storage_script.cpp.md)
**Tier floor** — T3: an abstract interface with one default algorithm

## Purpose

The *evaluation function* system answers questions about creatures and items with a number:
how healthy is this one, how good is that weapon, how likely am I to win this fight. The
answers feed a learned decision layer (see [`ef_pattern.cpp`](ef_pattern.cpp.md)) which is
the alife simulation's brain for offline decisions — what to carry, whom to attack, where to
go.

This is the interface. It is an abstract base with no implementation file, so the contract it
demands is the substance, and the discretization rule it supplies by default is the single
most consequential piece of code in the family.

## State

```text
RECORD BaseFunction                 # abstract
  min_result : real                 # the bottom of the declared range
  max_result : real                 # the top
  name       : text                 # how scripts and the pattern loader address it
  storage    : Storage              # the shared parameter block; required
```

**Invariant** — the range is *declared*, not measured. A function may return values outside
it; discretization clamps. Several functions rewrite their own maximum on each evaluation —
health, for instance, sets the maximum to the creature's own maximum health — so the range is
per-evaluation, not per-function. A rebuild must let a function adjust its own bounds.

**Invariant** — a function reads its inputs from the **shared parameter block**, not from
arguments. The caller fills in up to four slots — a member, an enemy, an item belonging to
each — and then asks a function for its value. This is a global-variable calling convention
and it is the reason the whole system is single-threaded and non-reentrant; see
[`ef_storage.h`](ef_storage.h.md).

## `ffGetValue`

**Contract** — abstract. Returns the function's real-valued answer for whatever is currently
in the parameter block. May adjust the declared range as a side effect. Implementors are in
[`ef_primary.cpp`](ef_primary.cpp.md) and [`ef_pattern.cpp`](ef_pattern.cpp.md).

## `dwfGetDiscreteValue`

**Contract** — maps the real answer onto `n` buckets, defaulting to two. Total; the clamps
make every input representable.

```text
FUNCTION discrete(n = 2) -> int
  v = value()
  IF v <= min_result THEN RETURN 0
  IF v >= max_result THEN RETURN n - 1
  RETURN round( (v - min_result) / (max_result - min_result) * (n - 1) )
```

**Invariants** — the interior is scaled by `n - 1`, not `n`, so the endpoints of the range
land exactly on bucket 0 and bucket `n-1` and the buckets are evenly spaced *including* the
ends. That is the right choice for a value that is being used as an index into a trained
table, and it is the convention the shipped trained data was built against. Using `n` instead
shifts every index and silently corrupts every learned lookup.

**Invariants** — the bucket count is supplied by the *caller*, because it is the caller's
table that is being indexed. The pattern functions pass the range of the variable their
trained data was built for; the direct callers pass two. A function therefore has no single
resolution.

**Notes** — several primary functions override this with a hand-written, deliberately
**non-linear** bucketing — maximum health and ammunition count both do. Those overrides
express "the interesting differences are not evenly spaced": the gap between thirty and fifty
health matters more than the gap between five hundred and seven hundred and fifty. A rebuild
must keep the specific break points, because the trained data was fitted against them.

## `ffGetMinResultValue` · `ffGetMaxResultValue` · `Name`

**Contract** — the declared range, read back so that a caller can size its own bucket count
from it, and the function's name, which is how a script and the trained-data loader address
it.

## `clsid_member` · `clsid_enemy` · `clsid_member_item` · `clsid_enemy_item`

**Contract** — the class identifiers of the four parameter slots, each resolved from whichever
of the two parameter blocks is currently filled. Implemented in
[`ef_primary.cpp`](ef_primary.cpp.md). Asking for a slot nothing was put in is a contract
violation.
