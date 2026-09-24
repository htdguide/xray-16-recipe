# src/editors/xrWeatherEditor/property_integer_limited.cpp

> A whole-number row that clamps to an authored range in both directions.

**Needs** — [`property_integer_limited.hpp`](property_integer_limited.hpp.md) · [`property_integer.hpp`](property_integer.hpp.md)
**Used by** — reached through its declarations in [`property_integer_limited.hpp`](property_integer_limited.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement of the accessor-bound whole-number adapter

## Purpose

The whole-number counterpart of [`property_float_limited.cpp`](property_float_limited.cpp.md).

## State

```text
RECORD ClampedIntegerProperty EXTENDS IntegerProperty
  min : int
  max : int            # invariant: max >= min, relied on and unchecked
```

## `GetValue` · `SetValue`

**Contract** — Clamp into `[min, max]` on read and on write. Out-of-range input is accepted and silently moved to the boundary rather than rejected.

**Notes** — Clamping the read means a document holding an out-of-range number displays in range and is corrected by the first edit; the reasoning is identical to the real-valued case and written out there. For whole numbers the clamp also does duty as bounds safety, since several of these ranges are the valid index span of something the engine indexes without checking.
