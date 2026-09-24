# src/editors/xrWeatherEditor/property_holder.cpp

> One document node as the editor sees it: the identity, the lifetime, and the rule that deleting a node from the grid deletes it from the document.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a managed object reachable through a native interface pointer, released on a schedule a collector does not choose

## Purpose

A property holder is the editor's handle on one node of the document being authored — a time-of-day frame, a weather set, one sub-record of either. The engine creates it, hands it the getters and setters for its own fields, and from then on the node is what the user manipulates.

The node is deliberately thin. It owns no document data. Everything editable is a binding back into engine storage, which is what makes the editor safe to run against a live engine: there is no second copy of the weather keyframes to keep in step.

## State

```text
RECORD PropertyHolder
  container     : PropertyContainer       # the managed grid-facing object; created
                                          # in the constructor, lives as long as the node
  display_name  : text                    # label when shown as a collection element;
                                          # copied from native text at construction
  collection    : optional<Collection>    # the collection this node is an element of
  engine        : Engine                  # invariant: non-none for the node's whole life
  holder        : optional<HolderOwner>   # the object the engine registered as owner
  disposing     : bool                    # guards against re-entering teardown
```

The container is created eagerly in the constructor, before any property is added, because the per-type `add_property` calls that follow all register into it and the engine issues them immediately after construction.

## `construct(engine, display_name, collection, holder)`

**Contract** — Creates the node and its empty container. The display name is copied out of engine-owned text at this moment; the node does not hold the engine's text alive. The collection and the owner may both be absent — a node that is not an element of anything and answers to no one is legitimate, and is how the three top-level frame holders are created. Allocates; does not block.

## `on_dispose`

**Contract** — Invoked when the user deletes this node through the grid. Removes the node from the document, not merely from the view. Idempotent by the disposing guard.

```text
FUNCTION on_dispose()
  IF disposing THEN RETURN

  index = collection.index_of(self)
  IF index < 0 THEN
    # the node was never inserted into its collection — it exists only as a
    # candidate the user has now discarded, so the collection destroys it outright
    collection.destroy(self)
    RETURN

  collection.erase(index)              # erase owns the destruction from here
```

**Invariants** — A node reaching this point belongs to a collection; a node with no collection has no deletion affordance and must not arrive here. The returned index, when non-negative, is within the collection's current size.

**Notes** — The two arms are not redundant. A collection creates candidate nodes before the user has committed them — the "add element" affordance builds a node and shows it — so a node can be fully constructed, addressable, and still absent from the collection's ordering. Asking the collection for the node's position is how the two states are told apart, and the collection is the only party that can answer, because the ordering is its data.

The disposing flag exists because two paths reach teardown: the user's explicit delete, and the runtime reclaiming the node later. Both must converge on exactly one removal. The decision a rebuild must reproduce is that the *first* path to arrive performs the work and the second becomes a no-op — not that any particular one of them is authoritative.

## `clear`

**Contract** — Empties the container of registered properties. The bindings are dropped; the engine data they pointed at is untouched. Used when a node is about to be re-populated with a different property set.

## `container` · `engine` · `holder` · `display_name` · `collection`

**Contract** — Each returns the correspondingly named field. `engine` requires the field to be set and is a programming error otherwise — no node is ever constructed without one.
