# src/editors/xrWeatherEditor/property_string_shared_str.hpp

> Declares the text adapter bound to a slot in the engine's interned-text store.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md)
**Used by** — [`property_file_name_value_shared_str.cpp`](property_file_name_value_shared_str.cpp.md) · [`property_file_name_value_shared_str.hpp`](property_file_name_value_shared_str.hpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md) · [`property_string_values_value_shared_str.cpp`](property_string_values_value_shared_str.cpp.md) · [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md) · [`property_string_values_value_shared_str_getter.cpp`](property_string_values_value_shared_str_getter.cpp.md) · [`property_string_values_value_shared_str_getter.hpp`](property_string_values_value_shared_str_getter.hpp.md)
**Tier floor** — T2: declares a managed object aliasing an engine-owned interned-text handle

## Purpose

Declares the surface implemented in [`property_string_shared_str.cpp`](property_string_shared_str.cpp.md).

## Exported units

- **`property_string_shared_str`** — a grid-editable text value aliased onto an interned-text slot the engine owns; the reference-bound counterpart of [`property_string`](property_string.hpp.md).
- **`GetValue` / `SetValue`** — read and write through the engine facade.
