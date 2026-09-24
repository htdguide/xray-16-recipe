# src/editors/xrWeatherEditor/property_float_reference.hpp

> Declares the reference-bound real-number property adapter.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_float_enum_value_reference.cpp`](property_float_enum_value_reference.cpp.md) · [`property_float_enum_value_reference.hpp`](property_float_enum_value_reference.hpp.md) · [`property_float_limited_reference.cpp`](property_float_limited_reference.cpp.md) · [`property_float_limited_reference.hpp`](property_float_limited_reference.hpp.md) · [`property_float_reference.cpp`](property_float_reference.cpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md)
**Tier floor** — T2: declares a managed object holding an alias into engine-owned storage

## Purpose

Declares the surface implemented in [`property_float_reference.cpp`](property_float_reference.cpp.md).

## Exported units

- **`property_float_reference`** — a grid-editable real bound directly to an engine field, with a nudge step; the reference-bound twin of [`property_float`](property_float.hpp.md).
- **`GetValue` / `SetValue` / `Increment`** — as in the accessor-bound adapter.
