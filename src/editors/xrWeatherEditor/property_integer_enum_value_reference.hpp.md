# src/editors/xrWeatherEditor/property_integer_enum_value_reference.hpp

> Declares the reference-bound whole-number adapter restricted to an authored set of named values.

**Needs** — [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_enum_value_reference.cpp`](property_integer_enum_value_reference.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_integer_enum_value_reference.cpp`](property_integer_enum_value_reference.cpp.md).

## Exported units

- **`property_integer_enum_value_reference`** — [`property_integer_reference`](property_integer_reference.hpp.md) restricted to a list of `(value, label)` choices.
- **`GetValue` / `SetValue`** — as in the accessor-bound twin.
