# src/editors/xrWeatherEditor/property_string_values_value_getter.hpp

> Declares the accessor-bound text adapter whose admissible set is regenerated on every query.

**Needs** — [`property_string.hpp`](property_string.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md)
**Tier floor** — T2: owns native callback objects that produce a list on demand

## Purpose

Declares the surface implemented in [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md).

## Exported units

- **`property_string_values_value_getter`** — [`property_string`](property_string.hpp.md) whose admissible set the engine rebuilds per query.
- **`values`** — rebuilds and returns the sequence.
