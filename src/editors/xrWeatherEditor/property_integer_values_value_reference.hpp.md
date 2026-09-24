# src/editors/xrWeatherEditor/property_integer_values_value_reference.hpp

> Declares the reference-bound index-into-a-fixed-label-list adapter.

**Needs** — [`property_integer_reference.hpp`](property_integer_reference.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_values_value_reference.cpp`](property_integer_values_value_reference.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_integer_values_value_reference.cpp`](property_integer_values_value_reference.cpp.md).

## Exported units

- **`property_integer_values_value_reference`** — the reference-bound twin of [`property_integer_values_value`](property_integer_values_value.hpp.md).
- **`GetValue` / `SetValue` / `collection`** — as in the accessor-bound twin.
