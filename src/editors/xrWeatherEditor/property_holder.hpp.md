# src/editors/xrWeatherEditor/property_holder.hpp

> Declares the editor's implementation of the engine's property-holder interface — one document node, presentable as a grid of editable properties.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — [`ide_impl.cpp`](ide_impl.cpp.md) · [`property_collection_base.cpp`](property_collection_base.cpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`property_collection_enumerator.cpp`](property_collection_enumerator.cpp.md) · [`property_collection_getter.hpp`](property_collection_getter.hpp.md) · [`property_container.cpp`](property_container.cpp.md) · [`property_holder.cpp`](property_holder.cpp.md) · [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md) · [`property_holder_collection.cpp`](property_holder_collection.cpp.md) · [`property_holder_color.cpp`](property_holder_color.cpp.md) · [`property_holder_container.cpp`](property_holder_container.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md) · [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md) · _and 4 more_
**Tier floor** — T2: it is the managed object the engine holds a native interface pointer to, so its identity must survive across the boundary

## Purpose

Declares the surface implemented in [`property_holder.cpp`](property_holder.cpp.md) (lifetime and identity) and in the seven per-type dispatch files ([`property_holder_boolean.cpp`](property_holder_boolean.cpp.md), [`property_holder_integer.cpp`](property_holder_integer.cpp.md), [`property_holder_float.cpp`](property_holder_float.cpp.md), [`property_holder_string.cpp`](property_holder_string.cpp.md), [`property_holder_color.cpp`](property_holder_color.cpp.md), [`property_holder_vec3f.cpp`](property_holder_vec3f.cpp.md), [`property_holder_container.cpp`](property_holder_container.cpp.md), [`property_holder_collection.cpp`](property_holder_collection.cpp.md)).

The split into one file per declared type is not arbitrary: the interface it satisfies is a wide family of overloads distinguished only by the type being bound, and each type needs its own set of value adapters, converter and editor. A rebuild whose interface dispatches on an explicit type tag rather than on overload resolution can collapse all eight into one.

## Exported units

- **`property_holder`** — the editor-side implementation of the engine's property-holder interface; owns one property container and knows which collection, if any, it is an element of.
- **`add_property`** (one per declared type and binding flavour) — registers one editable property with the container. Contracts in the per-type files.
- **`holder`** — returns the owner object the engine registered alongside this node.
- **`clear`** — drops every registered property.
- **`container`** — the managed grid-facing object this node presents as.
- **`engine`** — the engine facade this node reads shared text through.
- **`display_name`** — the label this node shows when it appears as an element of a collection.
- **`collection`** — the collection this node belongs to, or none.
- **`on_dispose`** — the user asked the grid to delete this node; described in [`property_holder.cpp`](property_holder.cpp.md).
