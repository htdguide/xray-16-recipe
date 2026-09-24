# src/editors/xrWeatherEditor/property_color.hpp

> Declares the callable-bound colour property.

**Needs** — [`property_color.cpp`](property_color.cpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md)
**Used by** — [`property_color.cpp`](property_color.cpp.md) · [`property_converter_color.cpp`](property_converter_color.cpp.md) · [`property_holder_color.cpp`](property_holder_color.cpp.md)
**Tier floor** — T1: declares a managed type owning two unmanaged callables.

## Purpose

The surface implemented in [`property_color.cpp`](property_color.cpp.md).

## The exported units

- **`property_color`** — a colour binding whose reads and writes go through a getter/setter pair.
