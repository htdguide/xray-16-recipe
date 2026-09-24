# src/editors/xrWeatherEditor/property_float_limited_reference.cpp

> The range-clamped real row, bound by alias instead of by callbacks.

**Needs** — [`property_float_limited_reference.hpp`](property_float_limited_reference.hpp.md) · [`property_float_reference.hpp`](property_float_reference.hpp.md)
**Used by** — reached through its declarations in [`property_float_limited_reference.hpp`](property_float_limited_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement of the reference-bound real adapter

## Purpose

Clamping applied to the reference-bound real. The clamp rules and their reasons are identical to [`property_float_limited.cpp`](property_float_limited.cpp.md); only the binding differs.

## State

```text
RECORD ClampedRealPropertyByReference EXTENDS RealPropertyByReference
  min : real
  max : real            # invariant: max >= min
```

## `GetValue` · `SetValue`

**Contract** — Clamp into `[min, max]` on read and on write, delegating the actual access to the aliased field.

**Notes** — That this file and [`property_float_limited.cpp`](property_float_limited.cpp.md) contain the same clamp twice is an artefact of the binding flavours being separate types. A rebuild that makes the binding a *field* of one real adapter rather than a choice of base collapses the whole doubled hierarchy — four real adapters instead of eight, and the same for whole numbers and text.
