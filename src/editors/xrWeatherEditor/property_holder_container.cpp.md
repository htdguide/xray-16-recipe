# src/editors/xrWeatherEditor/property_holder_container.cpp

> Registers one node as a nested property of another, which is how the document becomes a tree.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_property_container.hpp`](property_property_container.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: resolves an engine-side interface pointer to the concrete editor-side node

## Purpose

One registration overload: a property whose value is another property holder. This is the only structural composition in the document model — a weather keyframe that contains a sub-record shows the sub-record as an expandable row rather than flattening its fields into the parent.

## State

Stateless.

## `add_property` (nested node)

**Contract** — Takes the engine's interface handle on a child node, recovers the concrete editor-side node behind it, and registers the child's container as the value of a row on the parent. The recovery must succeed: a node reaching here was created by this editor, so a handle that does not resolve is a programming error and not a runtime condition. The child's lifetime is unaffected — the parent holds a reference, not ownership.

**Notes** — Downcasting from the engine's abstract handle to the editor's concrete node is where the two halves of the process stop being polite to each other. It is unavoidable here because the parent needs the child's *presentation* object, and presentation is deliberately absent from the abstract interface the engine sees. A rebuild can avoid it by having the engine hand over an opaque token the editor minted, rather than its own interface pointer; the decision to keep the interface narrow is worth preserving either way.
