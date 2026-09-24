# src/editors/xrWeatherEditor/property_vec3f_base.hpp

> Declares the vector property adapter's shared half: the three-component presentation value, the nested component rows, and the abstract whole-vector access an implementor must supply.

**Needs** — [`property_holder_include.hpp`](property_holder_include.hpp.md) · [`property_container_holder.hpp`](property_container_holder.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`property_converter_vec3f.cpp`](property_converter_vec3f.cpp.md) · [`property_vec3f.cpp`](property_vec3f.cpp.md) · [`property_vec3f.hpp`](property_vec3f.hpp.md) · [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md) · [`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md) · [`property_vec3f_reference.hpp`](property_vec3f_reference.hpp.md)
**Tier floor** — T2: declares a native trampoline object that keeps a managed object reachable

## Purpose

Declares the surface implemented in [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md), plus one value shape and one helper that have no implementation file of their own.

## Exported units

- **`Vec3f`** — the presentation layer's three-real value: what the grid parses, displays and passes around for a vector property. Distinct from the engine's vector record; the two are converted at every crossing.
- **`vec3f_components`** — the native trampoline that turns "set the x component" into a read-modify-write of the whole vector. Described in [`property_vec3f_base.cpp`](property_vec3f_base.cpp.md), where the reason it must be native is the point.
- **`property_vec3f_base`** — the abstract vector row: owns the nested container of three component rows, converts between the two vector shapes, and demands whole-vector read and write from its implementor.
- **`get_value_raw` / `set_value_raw`** — the two operations an implementor must supply. **This is the contract a rebuild must satisfy**: a vector property is defined entirely by the ability to read and write the vector as a whole. Everything else — components, text form, nudging — is derived from those two.
- **`x` / `y` / `z`** — set one component of the whole vector.
- **`GetValue` / `SetValue`** — the grid-facing pair; described in the implementation.
