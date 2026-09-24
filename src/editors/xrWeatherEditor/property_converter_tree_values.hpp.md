# src/editors/xrWeatherEditor/property_converter_tree_values.hpp

> Declares the converter whose only job is to refuse typed input.

**Needs** — _(none)_
**Used by** — [`property_converter_tree_values.cpp`](property_converter_tree_values.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T3: pure presentation; it never touches engine memory

## Purpose

Declares the converter implemented in [`property_converter_tree_values.cpp`](property_converter_tree_values.cpp.md).

## Exported units

- **`property_converter_tree_values`** — attached to a row whose value must come from its modal chooser; refuses every conversion into the row's value.
