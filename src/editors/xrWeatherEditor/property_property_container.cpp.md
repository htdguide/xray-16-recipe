# src/editors/xrWeatherEditor/property_property_container.cpp

> The row that makes the document a tree: its value is another node's whole property set.

**Needs** — [`property_property_container.hpp`](property_property_container.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a managed adapter holding a native pointer to an editor-side node

## Purpose

A nesting adapter with no value of its own. Registered by [`property_holder_container.cpp`](property_holder_container.cpp.md), it lets a keyframe present a sub-record as an expandable row rather than flattening the sub-record's fields into the parent.

## State

```text
RECORD NestedNodeProperty
  child : PropertyHolder        # referenced, not owned; outlives this adapter
```

## `GetValue`

**Contract** — Returns the child node's container. The grid recognises a container as a value and renders it as an expandable row whose children are the child's own properties. Cheap; no copy.

## `SetValue`

**Contract** — Unreachable. A nested node is edited through its own rows; there is no operation that replaces one wholesale. Reaching this is a programming error, not a user error, and it is treated as one.

**Notes** — Nesting by *reference to the child's presentation* rather than by copying the child's properties into the parent is what keeps a sub-record editable from more than one place at once — the same node can appear under two parents and both views stay in step, because there is only one node. It also means the parent must not own the child: ownership belongs to whoever created it, usually the engine.
