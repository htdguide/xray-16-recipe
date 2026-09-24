# src/editors/xrWeatherEditor/property_integer_limited_reference.cpp

> The range-clamped whole-number row, bound by alias instead of by callbacks.

**Needs** — [`property_integer_limited_reference.hpp`](property_integer_limited_reference.hpp.md) · [`property_integer_reference.hpp`](property_integer_reference.hpp.md)
**Used by** — reached through its declarations in [`property_integer_limited_reference.hpp`](property_integer_limited_reference.hpp.md); callers name that, not this file.
**Tier floor** — T2: a managed refinement of the reference-bound whole-number adapter

## Purpose

Clamping applied to the reference-bound whole number. Rules identical to [`property_integer_limited.cpp`](property_integer_limited.cpp.md).

## State

```text
RECORD ClampedIntegerPropertyByReference EXTENDS IntegerPropertyByReference
  min : int
  max : int            # invariant: max >= min
```

## `GetValue` · `SetValue`

**Contract** — Clamp into `[min, max]` on read and on write, accessing the aliased field directly.

**Notes** — The fourth copy of the same clamp in this directory. See [`property_float_limited_reference.cpp`](property_float_limited_reference.cpp.md) for the restructuring that removes all four.
