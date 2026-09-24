# src/editors/xrWeatherEditor/property_float_enum_value.hpp

> Declares the accessor-bound real adapter restricted to an authored set of named magnitudes.

**Needs** — [`property_float.hpp`](property_float.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_converter_float_enum.cpp`](property_converter_float_enum.cpp.md) · [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md).

## Exported units

- **`property_float_enum_value`** — [`property_float`](property_float.hpp.md) restricted to a list of `(magnitude, label)` choices.
- **`GetValue` / `SetValue`** — snap to a listed magnitude; accept a label.
- **`Increment`** — suppressed.
