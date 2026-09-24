# src/editors/xrWeatherEditor/property_string_values_value_getter.cpp

> A text row whose set of admissible values is asked for fresh every time, because it changes while the editor runs.

**Needs** — [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md) · [`property_string.hpp`](property_string.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md); callers name that, not this file.
**Tier floor** — T2: owns native callback objects and rebuilds a managed sequence from native text on every query

## Purpose

The live variant of [`property_string_values_value.cpp`](property_string_values_value.cpp.md). This is the shape behind the rows that offer "which weather set" and "which keyframe" — sets the user is editing at the same time as choosing from.

## State

```text
RECORD LiveRestrictedTextProperty EXTENDS TextProperty
  list_getter : callback() -> native array of text   # owned copy
  list_size   : callback() -> int                    # owned copy
                                                     # invariant: both describe the same
                                                     # array at the same instant
```

No cached sequence; a cache is what would go stale.

## `construct(getter, setter, list_getter, list_size)` · `release`

**Contract** — Takes private copies of both list callbacks alongside the base's value callbacks; frees all four exactly once. Release rule as in [`property_float.cpp`](property_float.cpp.md).

## `values`

**Contract** — Asks the engine for the array and its length, copies every entry into a fresh managed sequence, returns it. Allocates per call; retains nothing of the engine's.

```text
FUNCTION values() -> queue<text>
  source = list_getter()
  result = empty queue
  FOR EACH i IN 0 .. list_size() - 1
    result.enqueue(copy(source[i]))
  RETURN result
```

**Notes** — Unlike the whole-number live variant, nothing here has to clamp afterwards: the document stores the text, not a position, so a list that shrank leaves the stored value intact and merely unlisted. That is the payoff of storing text instead of an index, and it is why the asset-reference rows are text rows even where an index would be smaller in the file.
