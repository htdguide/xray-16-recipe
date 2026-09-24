# src/editors/xrWeatherEditor/property_file_name_value_shared_str.hpp

> Declares the interned-text row that names a file.

**Needs** — [`property_string_shared_str.hpp`](property_string_shared_str.hpp.md) · [`property_file_name_value_base.hpp`](property_file_name_value_base.hpp.md)
**Used by** — [`property_file_name_value_shared_str.cpp`](property_file_name_value_shared_str.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T2: a managed refinement over an engine-owned interned-text handle

## Purpose

Declares the surface implemented in [`property_file_name_value_shared_str.cpp`](property_file_name_value_shared_str.cpp.md).

## Exported units

- **`property_file_name_value_shared_str`** — [`property_string_shared_str`](property_string_shared_str.hpp.md) that also answers the file-chooser contract.
- **`default_extension` / `filter` / `initial_directory` / `title` / `remove_extension`** — the five chooser settings.
