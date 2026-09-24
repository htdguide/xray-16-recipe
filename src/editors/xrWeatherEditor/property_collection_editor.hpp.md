# src/editors/xrWeatherEditor/property_collection_editor.hpp

> Declares the collection row's dialog.

**Needs** — [`property_collection_editor.cpp`](property_collection_editor.cpp.md)
**Used by** — [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md)
**Tier floor** — T1: declares a managed type that drives an unmanaged collection interface.

## Purpose

The surface implemented in [`property_collection_editor.cpp`](property_collection_editor.cpp.md).

## The exported units

- **`property_collection_editor`** — the add/remove/reorder dialog for a collection row. Supplies the element type, the element factory, the per-element label, a dialog that repaints the rendered view when moved, and a re-entrancy path for nested collections.

It is attached to the collection bindings by declaration, in [`property_collection_base.hpp`](property_collection_base.hpp.md) and both of its subclasses.
