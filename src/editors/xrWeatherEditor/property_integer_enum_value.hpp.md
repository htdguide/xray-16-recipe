# src/editors/xrWeatherEditor/property_integer_enum_value.hpp

> Declares the accessor-bound whole-number adapter restricted to an authored set of named values.

**Needs** — [`property_integer.hpp`](property_integer.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_converter_integer_enum.cpp`](property_converter_integer_enum.cpp.md) · [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md).

## Exported units

- **`property_integer_enum_value`** — [`property_integer`](property_integer.hpp.md) restricted to a list of `(value, label)` choices.
- **`GetValue` / `SetValue`** — snap to a listed value; accept a label.
