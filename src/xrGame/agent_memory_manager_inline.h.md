# src/xrGame/agent_memory_manager_inline.h

> The bit-deletion primitive behind roster renumbering, plus the shared perception manager's list installation and access.

**Needs** — [`agent_memory_manager.h`](agent_memory_manager.h.md)
**Used by** — [`agent_memory_manager.cpp`](agent_memory_manager.cpp.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md)
**Tier floor** — T3: bit manipulation on a fixed-width mask.

## Purpose

Mostly accessors, but it holds one genuinely load-bearing operation: deleting a bit from a
squad mask and closing the gap. The rest is here for the original language's sake.

## `update_memory_mask`

**Contract** — Given a single-bit mask naming a departing roster position and a mask to
repair, rewrites the second so that bits above the departing position shift down one and
bits below keep their place. Pure, in-place, no branches.

```text
FUNCTION update_memory_mask(bit, current) -> mask
  above = complement of (bit OR (bit - 1))   # every position strictly higher
  below = bit - 1                            # every position strictly lower
  RETURN ((current AND above) SHIFTED RIGHT 1) OR (current AND below)
```

**Invariants** — `bit` has exactly one bit set. The mask type's width is fixed and the
complement is taken over that exact width, so a rebuild must not widen the type in one
place and not another.

**Notes** — This exists because a member's identity in a mask is its *index*, not a stable
identifier. The alternative design — a stable per-member identifier and a sparse set —
would make removal free and lookup expensive; the engine chose the opposite, and this
function is the price.

## List installation and access

**Contract** — `set_squad_objects` in three forms points the manager at externally owned
seen, heard and hit-by lists; `visibles`, `sounds` and `hits` hand them back, asserting
they were installed. Construction stores the owning squad manager and nothing else — the
three list references are deliberately uninitialized until installed, so using the manager
before installation is a detected programming error rather than a silent empty picture.
