# src/editors/xrWeatherEditor/property_vec3f.hpp

> Declares the accessor-bound vector property adapter.

**Needs** — [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md)
**Used by** — [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md) · [`property_converter_vec3f.hpp`](property_converter_vec3f.hpp.md) · [`property_holder_vec3f.cpp`](property_holder_vec3f.cpp.md) · [`property_vec3f.cpp`](property_vec3f.cpp.md)
**Tier floor** — T2: declares a managed object holding native callback objects

## Purpose

Declares the surface implemented in [`property_vec3f.cpp`](property_vec3f.cpp.md).

## Exported units

- **`property_vec3f`** — [`property_vec3f_base`](property_vec3f_base.hpp.md) bound to the engine by a whole-vector getter/setter pair.
- **`get_value_raw` / `set_value_raw`** — call the pair.
