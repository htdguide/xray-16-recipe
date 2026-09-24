# src/editors/xrWeatherEditor/property_converter_boolean_values.hpp

> Declares the two-label boolean renderer.

**Needs** — [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md)
**Used by** — [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md) · [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md)
**Tier floor** — T3: declares four overrides.

## Purpose

The surface implemented in [`property_converter_boolean_values.cpp`](property_converter_boolean_values.cpp.md).

## The exported units

- **`property_converter_boolean_values`** — offers a binding's two labels as an exclusive dropdown and renders the stored boolean as the matching one. Attached by name to the row when the engine describes a boolean with labels.
