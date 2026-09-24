# src/editors/xrWeatherEditor/property_integer_limited_reference.hpp

> Declares the range-clamped reference-bound whole-number adapter.

**Needs** — [`property_integer_reference.hpp`](property_integer_reference.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_limited_reference.cpp`](property_integer_limited_reference.cpp.md)
**Tier floor** — T2: a managed refinement of the reference-bound whole-number adapter

## Purpose

Declares the surface implemented in [`property_integer_limited_reference.cpp`](property_integer_limited_reference.cpp.md).

## Exported units

- **`property_integer_limited_reference`** — [`property_integer_reference`](property_integer_reference.hpp.md) with a minimum and a maximum.
- **`GetValue` / `SetValue`** — clamped read and clamped write.
