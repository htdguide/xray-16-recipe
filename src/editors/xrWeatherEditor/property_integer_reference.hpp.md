# src/editors/xrWeatherEditor/property_integer_reference.hpp

> Declares the reference-bound whole-number property adapter.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_holder_integer.cpp`](property_holder_integer.cpp.md) · [`property_integer_enum_value_reference.cpp`](property_integer_enum_value_reference.cpp.md) · [`property_integer_enum_value_reference.hpp`](property_integer_enum_value_reference.hpp.md) · [`property_integer_limited_reference.cpp`](property_integer_limited_reference.cpp.md) · [`property_integer_limited_reference.hpp`](property_integer_limited_reference.hpp.md) · [`property_integer_reference.cpp`](property_integer_reference.cpp.md) · [`property_integer_values_value_reference.cpp`](property_integer_values_value_reference.cpp.md) · [`property_integer_values_value_reference.hpp`](property_integer_values_value_reference.hpp.md) · [`property_integer_values_value_reference_getter.cpp`](property_integer_values_value_reference_getter.cpp.md) · [`property_integer_values_value_reference_getter.hpp`](property_integer_values_value_reference_getter.hpp.md)
**Tier floor** — T2: declares a managed object holding an alias into engine-owned storage

## Purpose

Declares the surface implemented in [`property_integer_reference.cpp`](property_integer_reference.cpp.md).

## Exported units

- **`property_integer_reference`** — a grid-editable whole number aliased onto an engine field; the reference-bound twin of [`property_integer`](property_integer.hpp.md).
- **`GetValue` / `SetValue`** — read and write the aliased field.
