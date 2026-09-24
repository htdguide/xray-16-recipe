# src/editors/xrWeatherEditor/property_collection.cpp

> The editable list, bound to one engine collection that will not move.

**Needs** — [`property_collection.hpp`](property_collection.hpp.md) · [`property_collection_base.cpp`](property_collection_base.cpp.md)
**Used by** — [`property_collection.hpp`](property_collection.hpp.md)
**Tier floor** — T1: it holds a pointer to an unmanaged collection it does not own.

## Purpose

The direct half of the collection binding pair. It answers [`property_collection_base`](property_collection_base.cpp.md)'s one open question — which collection — with a stored pointer.

## State

```text
RECORD DirectCollectionProperty EXTENDS CollectionProperty
  target : reference to EngineCollection      # not owned
```

**Invariants** — the referenced collection must outlive the property. As with every reference-bound form in this directory, nothing enforces it; the holder/object lifetime discipline does.

## Notes

The file is four lines of substance and exists only because the base class is abstract over this one choice. Use it when the collection is a stable member of an object — a keyframe list on a weather cycle, the flare list on a sun. Use [`property_collection_getter`](property_collection_getter.cpp.md) when which collection is current can change while the editor runs.
