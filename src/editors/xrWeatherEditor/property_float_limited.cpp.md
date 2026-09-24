# src/editors/xrWeatherEditor/property_float_limited.cpp

> A real row that clamps to an authored range on the way out as well as on the way in.

**Needs** — [`property_float_limited.hpp`](property_float_limited.hpp.md) · [`property_float.hpp`](property_float.hpp.md)
**Used by** — reached through its declarations in [`property_float_limited.hpp`](property_float_limited.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement of the accessor-bound real adapter

## Purpose

A real property whose valid values the engine declared at registration. Refines [`property_float.cpp`](property_float.cpp.md) by clamping in both directions.

## State

```text
RECORD ClampedRealProperty EXTENDS RealProperty
  min : real
  max : real            # invariant: max >= min, relied on and unchecked
```

## `GetValue`

**Contract** — Reads the bound value and clamps it into the range before the grid sees it.

```text
FUNCTION GetValue() -> real
  RETURN clamp(base.GetValue(), min, max)
```

**Notes** — Clamping the *read* is the interesting half, and it is a deliberate choice rather than symmetry. The engine's field can legitimately hold a value outside the declared range — the range is an authoring constraint the editor imposes, and the same data can be produced by an older file or by other engine code. Showing the true out-of-range value would let the user commit it back unchanged, entrenching it; showing the clamped value means the first edit of that row brings the quantity into range. The cost is that the row lies about the document until it is touched.

## `SetValue`

**Contract** — Clamps the incoming value into the range, then writes it through. Out-of-range input is never rejected — the grid accepts the keystroke and the value silently lands at the boundary.

**Notes** — Silent clamping over rejection: an authoring tool that refuses a keystroke has to explain itself, and the boundary value is almost always what the user meant. The clamp and the nudge step were chosen together — the step is a fraction of this same range ([`property_holder_float.cpp`](property_holder_float.cpp.md)) — so holding the nudge at a boundary parks the value there instead of running away.
