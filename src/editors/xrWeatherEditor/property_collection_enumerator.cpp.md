# src/editors/xrWeatherEditor/property_collection_enumerator.cpp

> Walks an engine collection for the grid, one element at a time, holding only a position.

**Needs** — [`property_collection_enumerator.hpp`](property_collection_enumerator.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_collection_enumerator.hpp`](property_collection_enumerator.hpp.md)
**Tier floor** — T1: it narrows an abstract engine holder to its concrete implementation on every step.

## Purpose

The grid's list editor iterates a collection rather than indexing it. This is the iterator, and like everything else in [the collection binding](property_collection_base.cpp.md) it stores no elements — only a position into the engine's live collection.

## State

```text
RECORD CollectionWalk
  target : reference to EngineCollection
  cursor : int         # -1 before the first element; == size() when exhausted
```

**Invariants** — the cursor begins at -1, so the first advance lands on element zero; it stops at the size and does not run past it. Reading the current element outside the valid range is an error, not an empty answer.

## `advance`

**Contract** — steps the cursor forward unless it already sits at the end, and reports whether it now points at an element.

```text
FUNCTION advance() -> bool
  IF cursor < size() THEN cursor = cursor + 1
  RETURN cursor != size()
```

**Notes** — the size is re-read on every step, so **the walk observes a collection that changes underneath it**. That is deliberate in the sense that everything in this directory reads through to the engine, and dangerous in the sense that nothing detects a concurrent modification. The grid's list editor mutates the collection between walks rather than during one, so it never bites. A rebuild whose iterators are invalidated by mutation gets a louder failure and should keep it.

## `current`

**Contract** — narrows the holder at the cursor to its concrete implementation and returns its container — the same mapping [`property_collection_base`](property_collection_base.cpp.md) makes. Fails if the cursor is before the first element or past the last.

## `reset`

**Contract** — returns the cursor to before the first element, so the collection can be walked again.
