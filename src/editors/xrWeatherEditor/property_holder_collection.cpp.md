# src/editors/xrWeatherEditor/property_holder_collection.cpp

> Registers an ordered, user-editable list of child nodes as a single property.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`property_collection_getter.hpp`](property_collection_getter.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: wraps a native collection interface for a managed list editor

## Purpose

Two registration overloads: a property whose value is a collection of child nodes, supplied either as a collection handle fixed at registration or as a callback that produces one on demand. This is how the weather document's variable-length parts — the keyframe list of a weather set — are authored: add, remove, reorder.

## State

Stateless.

## `add_property` (collection)

**Contract** — Registers a row whose declared type is the collection editor. No converter and no default value are supplied: a collection has no textual form and no meaningful empty default. Returns nothing.

**Notes** — The two overloads differ in *when* the collection is resolved, and the difference is the same one the live label lists make: a fixed handle is right when the owning record outlives the property registration, a callback is right when the collection the property refers to can be replaced wholesale — which happens when the user switches the frame being edited. The engine's own interface comments the callback form as probably removable; it is not, as long as anything the grid points at can be swapped underneath it.

All list mutation is the collection's own — insert, erase, create, destroy, and the label shown for each element. This file only hands the grid a way to reach it. That is what keeps element deletion correct: the grid's delete affordance reaches the element node, which asks its collection to remove it ([`property_holder.cpp`](property_holder.cpp.md)), so the document and the view can never disagree about the ordering.
