# src/editors/xrWeatherEditor/property_string_values_value.cpp

> A text row that also carries the list of values it admits, snapshotted when the row was built.

**Needs** — [`property_string_values_value.hpp`](property_string_values_value.hpp.md) · [`property_string.hpp`](property_string.hpp.md) · [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: copies a native text array into a managed sequence at construction

## Purpose

Reading and writing are unchanged from [`property_string.cpp`](property_string.cpp.md) — the document still stores the text itself. All this adapter adds is the published set of admissible values, which is what the converter turns into a drop-down and the tree chooser turns into a modal browser.

## State

```text
RECORD RestrictedTextProperty EXTENDS TextProperty
  admissible : queue<text>     # snapshot taken at construction; order is presentation only
```

## `construct(getter, setter, values, count)`

**Contract** — Copies `count` native strings into a managed sequence, in order. Allocates. The native array need not outlive the call.

## `values`

**Contract** — Returns the stored sequence.

**Notes** — Nothing checks that the value the document holds is one of the admissible ones, on read or on write. That is intentional and differs from every other restricted adapter here: the whole-number ones must snap, because an out-of-list number indexes nothing, whereas an out-of-list *text* still names something and may well be an asset the list simply failed to enumerate. Where the stricter behaviour is wanted, it is imposed at the presentation end by denying the row a text-parsing path at all ([`property_holder_string.cpp`](property_holder_string.cpp.md)), not by validating here.
