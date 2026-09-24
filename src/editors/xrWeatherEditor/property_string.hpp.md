# src/editors/xrWeatherEditor/property_string.hpp

> Declares the accessor-bound text property adapter.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_file_name_value.cpp`](property_file_name_value.cpp.md) · [`property_file_name_value.hpp`](property_file_name_value.hpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md) · [`property_string.cpp`](property_string.cpp.md) · [`property_string_values_value.cpp`](property_string_values_value.cpp.md) · [`property_string_values_value.hpp`](property_string_values_value.hpp.md) · [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md) · [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md)
**Tier floor** — T2: declares a managed object holding native callback objects

## Purpose

Declares the surface implemented in [`property_string.cpp`](property_string.cpp.md).

## Exported units

- **`property_string`** — a grid-editable text value bound to the engine by a getter/setter pair; the base of the chooser-restricted and file-name text adapters.
- **`GetValue` / `SetValue`** — read and write the bound text.
