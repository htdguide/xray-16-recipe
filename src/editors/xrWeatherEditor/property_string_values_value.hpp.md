# src/editors/xrWeatherEditor/property_string_values_value.hpp

> Declares the accessor-bound text adapter restricted to a fixed set of admissible values.

**Needs** — [`property_string.hpp`](property_string.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string_values_value.cpp`](property_string_values_value.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed sequence built from a native array

## Purpose

Declares the surface implemented in [`property_string_values_value.cpp`](property_string_values_value.cpp.md).

## Exported units

- **`property_string_values_value`** — [`property_string`](property_string.hpp.md) that also publishes the set of values it admits.
- **`values`** — returns the stored sequence; a field read, no computation.
