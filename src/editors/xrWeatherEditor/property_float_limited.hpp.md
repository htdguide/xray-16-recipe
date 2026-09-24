# src/editors/xrWeatherEditor/property_float_limited.hpp

> Declares the range-clamped accessor-bound real adapter.

**Needs** — [`property_float.hpp`](property_float.hpp.md)
**Used by** — [`property_color_base.cpp`](property_color_base.cpp.md) · [`property_float_limited.cpp`](property_float_limited.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md) · [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md)
**Tier floor** — T2: a managed refinement of the accessor-bound real adapter

## Purpose

Declares the surface implemented in [`property_float_limited.cpp`](property_float_limited.cpp.md).

## Exported units

- **`property_float_limited`** — [`property_float`](property_float.hpp.md) with a minimum and a maximum.
- **`GetValue` / `SetValue`** — clamped read and clamped write.
