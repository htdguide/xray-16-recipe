# src/editors/xrWeatherEditor/property_collection_converter.hpp

> Declares the collection row's text rendering.

**Needs** — [`property_collection_converter.cpp`](property_collection_converter.cpp.md)
**Used by** — [`property_collection_base.hpp`](property_collection_base.hpp.md) · [`property_collection_converter.cpp`](property_collection_converter.cpp.md)
**Tier floor** — T3: declares two overrides.

## Purpose

The surface implemented in [`property_collection_converter.cpp`](property_collection_converter.cpp.md).

## The exported units

- **`property_collection_converter`** — answers "can this value become text" (yes, and only text) and "what text" (a fixed ellipsis). Attached to the collection bindings by declaration, in [`property_collection_base.hpp`](property_collection_base.hpp.md) and both of its subclasses.
