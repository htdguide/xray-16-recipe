# src/editors/xrWeatherEditor/property_integer_limited.hpp

> Declares the range-clamped accessor-bound whole-number adapter.

**Needs** — [`property_integer.hpp`](property_integer.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_limited.cpp`](property_integer_limited.cpp.md)
**Tier floor** — T2: a managed refinement of the accessor-bound whole-number adapter

## Purpose

Declares the surface implemented in [`property_integer_limited.cpp`](property_integer_limited.cpp.md).

## Exported units

- **`property_integer_limited`** — [`property_integer`](property_integer.hpp.md) with a minimum and a maximum.
- **`GetValue` / `SetValue`** — clamped read and clamped write.
