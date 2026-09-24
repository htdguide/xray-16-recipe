# src/xrCore/_vector4.h

> The 4-vector: shader constants, planes, and quantities that must reach the graphics device as a four-float register.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`math_constants.h`](math_constants.h.md)
**Used by** — [`_matrix.h`](_matrix.h.md) · [`vector.h`](vector.h.md) · [`xr_ini.h`](xr_ini.h.md)
**Tier floor** — T1: it is four consecutive floats, which is exactly one constant register on every graphics backend; constant buffers are filled by copying arrays of it.

## Purpose

Four components, templated and instantiated for 32-bit float and 32-bit signed integer. Its dominant use is as the unit a shader constant buffer is built from — the renderer packs parameters into these and uploads them as raw bytes — and secondarily as a plane (normal in the first three components, distance in the fourth).

## State

```text
RECORD Vector4 of T
  x : T
  y : T
  z : T
  w : T
```

Four components, in that order, no padding. Indexed access aliases the names.

**Invariants** — The fourth component's meaning is the caller's: it is a homogeneous coordinate in some uses, a plane distance in others, an alpha channel in others. Nothing in the type knows. Notably, **the set and subtract and multiply operations default the fourth component to 1**, not to 0 — the homogeneous-point convention — so an operation written with three arguments silently makes a point, not a direction.

## Operations

| Group | Operations |
|---|---|
| assignment | from four components (fourth defaulting to one), or from another 4-vector |
| arithmetic | add, subtract, multiply, divide — against a vector or a scalar, in place or into this |
| similarity | componentwise within an epsilon |
| length | squared magnitude and magnitude over **all four** components |
| normalization | normalize over all four components |
| plane normalization | normalize so the **first three** components are unit length, scaling the fourth with them — the operation that makes a plane equation canonical |
| interpolation | linear interpolation of all four components |

## Notes

The distinction between the two normalizations is the only subtlety in the file and it is load-bearing: normalizing a plane over four components produces a plane whose distance term is meaningless, and the engine uses both operations on the same type. A rebuild should consider making planes a distinct type; the engine did not, and the resulting confusion is worth knowing about.

The validity predicate requires all four components to be finite and normal.

The indexed accessor has the same constness defect as the 2-vector — it hands out a mutable reference from a constant vector.
