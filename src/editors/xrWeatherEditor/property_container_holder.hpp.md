# src/editors/xrWeatherEditor/property_container_holder.hpp

> An empty marker: "this object owns a property container".

**Needs** — [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_color_base.hpp`](property_color_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md)
**Tier floor** — T3: a type tag with no members.

## Purpose

It declares an interface with no operations at all.

Its only job is to give [`property_container`](property_container.cpp.md) a typed slot for "the object that owns me", which composite bindings then narrow to their own concrete type to reach themselves. [`property_color_base`](property_color_base.cpp.md) is the one user: its three component rows live in a container of their own, and the [colour renderer](property_converter_color.cpp.md) reaches the colour value by asking that container for its owner and narrowing it back.

## State

`Stateless.`

## `property_container_holder`

**Contract** — an object that satisfies this marker may be stored as a container's owner and narrowed back to its own type by whatever needs to reach it. It demands nothing of an implementor, which is exactly the weakness noted below.

## Notes

An empty marker is a weak way to express this — the narrowing is unchecked in release builds, and any owner type would satisfy the declaration. What the file really encodes is a decision worth keeping: **a container knows what owns it, so a nested value's rows can find their way back to the value.** Without that link, the colour renderer would have three component rows and no colour.

A rebuild gives the container a typed back-reference to the composite binding, and the marker disappears.
