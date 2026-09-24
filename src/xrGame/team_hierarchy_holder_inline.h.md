# src/xrGame/team_hierarchy_holder_inline.h

> Construction and the two upward accessors.

**Needs** — [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md)
**Used by** — [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md)
**Tier floor** — T3: pre-sizing an array and reading two fields.

## Purpose

Three small members kept out of the header. One carries a decision.

## `construct(parent)`

**Contract** — binds the parent team registry, which is required, and **pre-sizes the squad
array to its full 256 slots with every slot empty**.

```text
FUNCTION construct(parent)
  REQUIRE parent EXISTS
  team   := parent
  squads := 256 empty slots
```

**Invariants** — the pre-sizing is what makes a squad identifier an index. Without it the
lookup in [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md) would have to grow the
array, which would invalidate references handed out earlier — and those references are held
by creatures for their whole lives.

## `team()` / `squads()`

**Contract** — the parent registry, and the whole slot array. The array is handed out
including its empty slots; a caller walking it must skip them.
