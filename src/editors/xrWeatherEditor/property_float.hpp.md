# src/editors/xrWeatherEditor/property_float.hpp

> Declares the accessor-bound real-number property adapter.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md)
**Used by** — [`property_float.cpp`](property_float.cpp.md) · [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md) · [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) · [`property_float_limited.cpp`](property_float_limited.cpp.md) · [`property_float_limited.hpp`](property_float_limited.hpp.md) · [`property_holder_float.cpp`](property_holder_float.cpp.md) · [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md)
**Tier floor** — T2: declares a managed object holding native callback objects

## Purpose

Declares the surface implemented in [`property_float.cpp`](property_float.cpp.md).

## Exported units

- **`property_float`** — a grid-editable real bound to the engine by a getter/setter pair, with a nudge step. Also answers the grid's "this value can be incremented" protocol, which is what gives the row its drag-and-spin behaviour.
- **`GetValue` / `SetValue`** — read and write the bound real.
- **`Increment`** — nudge by a multiple of the step.
