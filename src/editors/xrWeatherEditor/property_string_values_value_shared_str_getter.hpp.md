# src/editors/xrWeatherEditor/property_string_values_value_shared_str_getter.hpp

> Declares the interned-text adapter whose admissible set is regenerated on every query.

**Needs** — [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string_values_value_shared_str_getter.cpp`](property_string_values_value_shared_str_getter.cpp.md)
**Tier floor** — T2: owns native callback objects and aliases an engine-owned interned-text handle

## Purpose

Declares the surface implemented in [`property_string_values_value_shared_str_getter.cpp`](property_string_values_value_shared_str_getter.cpp.md).

## Exported units

- **`property_string_values_value_shared_str_getter`** — [`property_string_shared_str`](property_string_shared_str.hpp.md) whose admissible set the engine rebuilds per query.
- **`values`** — rebuilds and returns the sequence.
