# src/editors/xrWeatherEditor/property_float_limited_reference.hpp

> Declares the range-clamped reference-bound real adapter.

**Needs** — [`property_float_reference.hpp`](property_float_reference.hpp.md)
**Used by** — [`property_float_limited_reference.cpp`](property_float_limited_reference.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md)
**Tier floor** — T2: a managed refinement of the reference-bound real adapter

## Purpose

Declares the surface implemented in [`property_float_limited_reference.cpp`](property_float_limited_reference.cpp.md).

## Exported units

- **`property_float_limited_reference`** — [`property_float_reference`](property_float_reference.hpp.md) with a minimum and a maximum.
- **`GetValue` / `SetValue`** — clamped read and clamped write.
