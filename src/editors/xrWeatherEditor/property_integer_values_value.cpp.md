# src/editors/xrWeatherEditor/property_integer_values_value.cpp

> A whole number that is a position in a label list, with the list fixed when the row was built.

**Needs** — [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md) · [`property_integer.hpp`](property_integer.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md)
**Used by** — reached through its declarations in [`property_integer_values_value.hpp`](property_integer_values_value.hpp.md); callers name that, not this file.
**Tier floor** — T2: copies a native text array into a managed list at construction

## Purpose

The index-selection shape: the document stores `2`, the user sees the third label. Distinct from the named-value shape ([`property_integer_enum_value.cpp`](property_integer_enum_value.cpp.md)), where the document stores a number that means something on its own.

## State

```text
RECORD IndexSelectionProperty EXTENDS IntegerProperty
  labels : list<text>       # invariant: non-empty
                            # invariant: this exact ordering is what stored indices mean
```

## `construct(getter, setter, labels, count)`

**Contract** — Copies `count` labels into a managed list, in order. Allocates. The native array need not outlive the call.

## `GetValue`

**Contract** — Reads the stored index and clamps it into the list's bounds before the grid sees it.

```text
FUNCTION GetValue() -> int
  RETURN clamp(base.GetValue(), 0, labels.size - 1)
```

**Notes** — Clamping rather than failing: a document written against a longer list — an older build, a different installation — otherwise selects nothing and the row renders empty. Clamping puts the selection at the last label, which is wrong but visible and editable. The trade is the same one the range-clamped adapters make, with more at stake, because here the index *is* the meaning.

## `SetValue`

**Contract** — The grid supplies the label the user picked; stores its position. A label that is not in the list is a programming error — the grid only ever offers labels this adapter published — and is not handled.

## `collection`

**Contract** — Returns the stored list. Cheap; the same object every call.
