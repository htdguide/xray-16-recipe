# src/editors/xrWeatherEditor/property_boolean_values_value.hpp

> Declares the two-label boolean property.

**Needs** — [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md) · [`property_boolean.hpp`](property_boolean.hpp.md)
**Used by** — [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md) · [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md) · [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md)
**Tier floor** — T1: declares a managed type over two unmanaged callables and a label pair.

## Purpose

The surface implemented in [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md).

## The exported units

- **`property_boolean_values_value`** — a callable-bound boolean presented as a choice between two named values. Its label list is public, because [`property_converter_boolean_values`](property_converter_boolean_values.hpp.md) reads it to render the current value and to offer the dropdown.
