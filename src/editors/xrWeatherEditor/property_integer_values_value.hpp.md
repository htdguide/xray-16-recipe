# src/editors/xrWeatherEditor/property_integer_values_value.hpp

> Declares the accessor-bound index-into-a-fixed-label-list adapter.

**Needs** — [`property_integer.hpp`](property_integer.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md)
**Tier floor** — T2: a managed refinement holding a managed list built from a native array

## Purpose

Declares the surface implemented in [`property_integer_values_value.cpp`](property_integer_values_value.cpp.md).

## Exported units

- **`property_integer_values_value`** — [`property_integer`](property_integer.hpp.md) whose stored number is a position in a label list fixed at registration.
- **`GetValue` / `SetValue`** — clamped index out, label in.
- **`collection`** — the label list, for the converter.
