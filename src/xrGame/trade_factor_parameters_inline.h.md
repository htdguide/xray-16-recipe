# src/xrGame/trade_factor_parameters_inline.h

> The four operations of the per-section multiplier table.

**Needs** — [`trade_factor_parameters.h`](trade_factor_parameters.h.md)
**Used by** — [`trade_factor_parameters.h`](trade_factor_parameters.h.md)
**Tier floor** — T3: a keyed table.

## Purpose

The body of [`trade_factor_parameters.h`](trade_factor_parameters.h.md). A rebuild uses its
language's map and deletes this file.

## `clear` / `enable(section, factors)` / `enabled(section)` / `factors(section)`

**Contract** — empty the table; insert a section's pair, which must not already be present;
report whether a section has one; and return it, where the caller has established that it
does.

**Notes** — `factors` fails rather than returning a default for an absent section, which
forces the two-step "ask, then read" at every call site. That is what makes the fallback
chain in [`trade_parameters_inline.h`](trade_parameters_inline.h.md) explicit: each level
tests before it reads, and the reader can see exactly which level supplied the answer. A
rebuild returning an optional expresses the same thing more directly.
