# src/editors/xrWeatherEditor/property_collection.hpp

> Declares the directly bound editable list.

**Needs** — [`property_collection.cpp`](property_collection.cpp.md) · [`property_collection_base.hpp`](property_collection_base.hpp.md)
**Used by** — [`property_collection.cpp`](property_collection.cpp.md) · [`property_collection_editor.cpp`](property_collection_editor.cpp.md) · [`property_holder_collection.cpp`](property_holder_collection.cpp.md)
**Tier floor** — T1: declares a managed type holding an unmanaged pointer.

## Purpose

The surface implemented in [`property_collection.cpp`](property_collection.cpp.md).

## The exported units

- **`property_collection`** — an editable list over one fixed engine collection. Repeats the renderer and editor attachment from [`property_collection_base`](property_collection_base.hpp.md), because the grid reads those off the value's own type rather than inheriting them.
