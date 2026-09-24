# src/editors/xrWeatherEditor/property_integer.hpp

> Declares the accessor-bound whole-number property adapter.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer.cpp`](property_integer.cpp.md) · [`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md) · [`property_integer_enum_value.hpp`](property_integer_enum_value.hpp.md) · [`property_integer_limited.cpp`](property_integer_limited.cpp.md) · [`property_integer_limited.hpp`](property_integer_limited.hpp.md) · [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md) · [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md) · [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md) · [`property_integer_values_value_getter.hpp`](property_integer_values_value_getter.hpp.md)
**Tier floor** — T2: declares a managed object holding native callback objects

## Purpose

Declares the surface implemented in [`property_integer.cpp`](property_integer.cpp.md).

## Exported units

- **`property_integer`** — a grid-editable whole number bound to the engine by a getter/setter pair; the base of every whole-number adapter in this directory.
- **`GetValue` / `SetValue`** — read and write the bound number.
