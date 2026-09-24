# src/editors/xrWeatherEditor/property_file_name_value.hpp

> Declares the accessor-bound text row that names a file.

**Needs** — [`property_string.hpp`](property_string.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md)
**Used by** — [`property_editor_file_name.cpp`](property_editor_file_name.cpp.md) · [`property_file_name_value.cpp`](property_file_name_value.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T2: a managed refinement of the accessor-bound text adapter

## Purpose

Declares the surface implemented in [`property_file_name_value.cpp`](property_file_name_value.cpp.md).

## Exported units

- **`property_file_name_value`** — [`property_string`](property_string.hpp.md) that also answers the file-chooser contract.
- **`default_extension` / `filter` / `initial_directory` / `title` / `remove_extension`** — the five chooser settings, each a stored field.
