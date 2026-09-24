# src/editors/xrWeatherEditor/property_string_values_value_shared_str.hpp

> Declares the interned-text adapter restricted to a fixed set of admissible values.

**Needs** — [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string_values_value_shared_str.cpp`](property_string_values_value_shared_str.cpp.md)
**Tier floor** — T2: a managed refinement over an engine-owned interned-text handle

## Purpose

Declares the surface implemented in [`property_string_values_value_shared_str.cpp`](property_string_values_value_shared_str.cpp.md).

## Exported units

- **`property_string_values_value_shared_str`** — [`property_string_shared_str`](property_string_shared_str.hpp.md) that also publishes the set of values it admits.
- **`values`** — returns the stored sequence.
