# src/editors/xrWeatherEditor/property_integer_values_value_reference_getter.hpp

> Declares the reference-bound index adapter whose label list is regenerated on every query.

**Needs** — [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_values_value_reference_getter.cpp`](property_integer_values_value_reference_getter.cpp.md)
**Tier floor** — T2: owns native callback objects and an alias into engine-owned storage

## Purpose

Declares the surface implemented in [`property_integer_values_value_reference_getter.cpp`](property_integer_values_value_reference_getter.cpp.md).

## Exported units

- **`property_integer_values_value_reference_getter`** — the reference-bound twin of [`property_integer_values_value_getter`](property_integer_values_value_getter.hpp.md).
- **`GetValue` / `SetValue` / `collection`** — as in the accessor-bound twin.
