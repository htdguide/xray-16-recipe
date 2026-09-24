# src/editors/xrWeatherEditor/property_integer_values_value_getter.hpp

> Declares the accessor-bound index adapter whose label list is regenerated on every query.

**Needs** — [`property_integer.hpp`](property_integer.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md)
**Tier floor** — T2: owns native callback objects that produce a list on demand

## Purpose

Declares the surface implemented in [`property_integer_values_value_getter.cpp`](property_integer_values_value_getter.cpp.md).

## Exported units

- **`property_integer_values_value_getter`** — [`property_integer`](property_integer.hpp.md) whose stored number indexes a list the engine rebuilds per query.
- **`GetValue` / `SetValue`** — clamped index out, label in.
- **`collection`** — rebuilds and returns the list.
