# src/editors/xrWeatherEditor/property_collection_getter.cpp

> The editable list, where *which* collection is a question asked fresh on every access.

**Needs** — [`property_collection_getter.hpp`](property_collection_getter.hpp.md) · [`property_collection_base.cpp`](property_collection_base.cpp.md)
**Used by** — [`property_collection_getter.hpp`](property_collection_getter.hpp.md)
**Tier floor** — T1: it owns an unmanaged callable from a managed object.

## Purpose

The deferred half of the collection binding pair. Some collection rows do not name a fixed list: the set of keyframes shown in the grid is the *current* weather cycle's, and the current cycle changes while the editor runs. Binding a pointer would pin the grid to whichever cycle was current when the row was described.

## State

```text
RECORD DeferredCollectionProperty EXTENDS CollectionProperty
  which : callable() -> EngineCollection     # owned copy of the engine's callable
```

**Invariants** — the callable is copied at construction and released exactly once, on either disposal path. It is invoked on **every** collection operation — every count, every element read, every insert — and never cached.

## Notes

Never caching is the decision, and it is the same pull-not-push rule the timeline and the property grid follow: the answer is asked for at the moment it is needed, so it cannot be stale. The cost is a callable invocation per list operation, which for lists of tens is nothing.

A rebuild expresses both halves of this pair as one binding over a closure, and the direct form becomes a closure that returns a captured reference. The split into two types is an artifact of not wanting a callable indirection in the common case.
