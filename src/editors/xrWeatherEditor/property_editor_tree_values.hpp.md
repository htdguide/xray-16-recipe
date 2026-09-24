# src/editors/xrWeatherEditor/property_editor_tree_values.hpp

> Declares the modal tree chooser a text row opens to pick from a long, hierarchical list.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`window_tree_values.h`](window_tree_values.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T3: pure presentation; text crosses to the row through its own interface

## Purpose

Declares the value editor implemented in [`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md).

## Exported units

- **`property_editor_tree_values`** — holds one reusable chooser window and drives it from the row's admissible set.
- **`GetEditStyle` / `EditValue`** — the modal chooser.
