# src/editors/xrWeatherEditor/property_container_converter.hpp

> Declares the nested-object renderer.

**Needs** — [`property_container_converter.cpp`](property_container_converter.cpp.md)
**Used by** — [`property_container.cpp`](property_container.cpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_container_converter.cpp`](property_container_converter.cpp.md)
**Tier floor** — T3: declares four overrides.

## Purpose

The surface implemented in [`property_container_converter.cpp`](property_container_converter.cpp.md).

## The exported units

- **`property_container_converter`** — extends the grid's expandable-object renderer. Answers that a container has sub-rows (always), supplies them in the engine's declaration order, and renders the collapsed row as an ellipsis. Attached to [`property_container`](property_container.hpp.md) by declaration.
