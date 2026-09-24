# src/editors/xrWeatherEditor/property_property_container.hpp

> Declares the adapter that makes one node's value be another node's grid presentation.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md)
**Used by** — [`property_holder_container.cpp`](property_holder_container.cpp.md) · [`property_property_container.cpp`](property_property_container.cpp.md)
**Tier floor** — T2: a managed adapter holding a native pointer to an editor-side node

## Purpose

Declares the surface implemented in [`property_property_container.cpp`](property_property_container.cpp.md).

## Exported units

- **`property_property_container`** — a grid row whose value is a nested node's container.
- **`GetValue`** — the child's container.
- **`SetValue`** — never legal.
