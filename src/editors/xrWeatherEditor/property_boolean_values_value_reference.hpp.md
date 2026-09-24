# src/editors/xrWeatherEditor/property_boolean_values_value_reference.hpp

> Declares the field-bound two-label boolean property.

**Needs** — [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md) · [`property_boolean_reference.hpp`](property_boolean_reference.hpp.md)
**Used by** — [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md) · [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md)
**Tier floor** — T1: declares a managed type holding a reference to an unmanaged field.

## Purpose

The surface implemented in [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md).

## The exported units

- **`property_boolean_values_value_reference`** — a field-bound boolean presented as a choice between two named values. Its label list is public for [the converter](property_converter_boolean_values.hpp.md) to read.
