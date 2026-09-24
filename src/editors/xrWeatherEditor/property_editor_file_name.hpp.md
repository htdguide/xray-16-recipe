# src/editors/xrWeatherEditor/property_editor_file_name.hpp

> Declares the file chooser a text row opens to pick an asset.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T3: pure presentation; text crosses to the row through its own interface

## Purpose

Declares the value editor implemented in [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md).

## Exported units

- **`property_editor_file_name`** — holds one reusable file dialog and drives it from the row's chooser settings.
- **`GetEditStyle` / `EditValue`** — the modal chooser.
