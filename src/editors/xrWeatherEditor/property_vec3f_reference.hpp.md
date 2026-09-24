# src/editors/xrWeatherEditor/property_vec3f_reference.hpp

> Declares the grid row for a three-component vector the engine owns, edited in place.

**Needs** — [`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md) · [`property_vec3f_base.hpp`](property_vec3f_base.hpp.md)
**Used by** — [`property_holder_vec3f.cpp`](property_holder_vec3f.cpp.md) · [`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md)
**Tier floor** — T1: it holds a direct reference into a native record from the managed side.

## Purpose

Declares the surface implemented in
[`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md). One of a family of row
types, split by *how the value is reached*: this one holds a reference to the engine's own
field, its sibling holds an accessor pair.

## State

```text
RECORD Vec3fReferenceRow EXTENDS Vec3fRow
  target : reference to a three-component vector owned by the engine
```

**Invariants** — the referenced vector must outlive this row. Nothing enforces it; see the
implementation.

## Exported units

- **construct** — bind to an engine-owned vector.
- **read / write the raw value** — the two operations the base row needs.
- **release** — drop the binding.
