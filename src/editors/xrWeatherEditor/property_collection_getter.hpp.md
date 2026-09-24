# src/editors/xrWeatherEditor/property_collection_getter.hpp

> Declares the deferred editable list.

**Needs** — [`property_collection_getter.cpp`](property_collection_getter.cpp.md) · [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`property_holder.hpp`](property_holder.hpp.md)
**Used by** — [`property_collection_getter.cpp`](property_collection_getter.cpp.md) · [`property_holder_collection.cpp`](property_holder_collection.cpp.md)
**Tier floor** — T1: declares a managed type owning an unmanaged callable.

## Purpose

The surface implemented in [`property_collection_getter.cpp`](property_collection_getter.cpp.md).

## The exported units

- **`property_collection_getter`** — an editable list whose backing collection is fetched through a callable on every access. Repeats the renderer and editor attachment from [`property_collection_base`](property_collection_base.hpp.md), for the same reason [`property_collection`](property_collection.hpp.md) does.
