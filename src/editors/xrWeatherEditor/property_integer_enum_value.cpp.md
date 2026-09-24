# src/editors/xrWeatherEditor/property_integer_enum_value.cpp

> A whole number that may only take one of an authored set of values, chosen by name.

**Needs** — [`property_integer_enum_value.hpp`](property_integer_enum_value.hpp.md) · [`property_integer.hpp`](property_integer.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: copies a native `(value, label)` array into a managed list at construction

## Purpose

The whole-number counterpart of [`property_float_enum_value.cpp`](property_float_enum_value.cpp.md), and the one that matters more: whole-number modes in the weather document — blend styles, sound set kinds — are stored as the authored number, not as a list position, so this adapter is what keeps the stored number stable when the list is reordered.

## State

```text
RECORD IntegerChoiceProperty EXTENDS IntegerProperty
  choices : list<Pair<int, text>>     # invariant: non-empty; entry 0 is the fallback
                                      # invariant: values are distinct
```

## `construct(getter, setter, choices, count)`

**Contract** — Copies `count` `(value, label)` pairs into a managed list, converting each label. Allocates. The native array need not outlive the call.

## `GetValue`

**Contract** — Reads the bound number; returns it if it appears in the list, and the first listed value otherwise.

**Notes** — Unlike the real-valued twin the comparison here is exact and exactly right, so the fallback fires only when the document genuinely holds a value the list does not name. Falling back to the first choice rather than surfacing the mismatch means such a value is quietly rewritten the next time the user touches the row — a real data-loss path, and the honest trade the original made for a grid with no error channel.

## `SetValue`

**Contract** — The grid supplies the label the user picked; writes the matching value, or the first choice's value when nothing matches.

**Notes** — Read yields a value, write takes a label. The mapping from value to label for display belongs to [`property_converter_integer_enum.cpp`](property_converter_integer_enum.cpp.md); this adapter holds the list because it is the party that owns the binding.
