# src/editors/xrWeatherEditor/property_float_enum_value_reference.hpp

> Declares the reference-bound real adapter restricted to an authored set of named magnitudes.

**Needs** — [`property_float_reference.hpp`](property_float_reference.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_float_enum_value_reference.cpp`](property_float_enum_value_reference.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_float_enum_value_reference.cpp`](property_float_enum_value_reference.cpp.md).

## Exported units

- **`property_float_enum_value_reference`** — [`property_float_reference`](property_float_reference.hpp.md) restricted to a list of `(magnitude, label)` choices.
- **`GetValue` / `SetValue` / `Increment`** — as in the accessor-bound twin.
