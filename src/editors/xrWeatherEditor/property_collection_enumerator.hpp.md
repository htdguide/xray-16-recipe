# src/editors/xrWeatherEditor/property_collection_enumerator.hpp

> Declares the collection walk.

**Needs** — [`property_collection_enumerator.cpp`](property_collection_enumerator.cpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_collection_base.cpp`](property_collection_base.cpp.md) · [`property_collection_enumerator.cpp`](property_collection_enumerator.cpp.md)
**Tier floor** — T1: declares a managed type holding an unmanaged collection pointer.

## Purpose

The surface implemented in [`property_collection_enumerator.cpp`](property_collection_enumerator.cpp.md).

## The exported units

- **`property_collection_enumerator`** — a forward walk over an engine collection, yielding each element's container. `reset`, `advance`, `current`.
