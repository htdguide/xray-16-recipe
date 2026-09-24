# src/editors/xrWeatherEditor/property_string_values_value_base.hpp

> The contract a converter or a chooser relies on to ask a text row which values it may take.

**Needs** — _(none)_
**Used by** — [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md) · [`property_converter_string_values.hpp`](property_converter_string_values.hpp.md) · [`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md) · [`property_editor_tree_values.hpp`](property_editor_tree_values.hpp.md) · [`property_string_values_value.cpp`](property_string_values_value.cpp.md) · [`property_string_values_value.hpp`](property_string_values_value.hpp.md) · [`property_string_values_value_getter.cpp`](property_string_values_value_getter.cpp.md) · [`property_string_values_value_getter.hpp`](property_string_values_value_getter.hpp.md) · [`property_string_values_value_shared_str.cpp`](property_string_values_value_shared_str.cpp.md) · [`property_string_values_value_shared_str.hpp`](property_string_values_value_shared_str.hpp.md) · [`property_string_values_value_shared_str_getter.cpp`](property_string_values_value_shared_str_getter.cpp.md) · [`property_string_values_value_shared_str_getter.hpp`](property_string_values_value_shared_str_getter.hpp.md) · [`window_tree_values.cpp`](window_tree_values.cpp.md) · [`window_tree_values.h`](window_tree_values.h.md)
**Tier floor** — T3: a pure presentation-side capability declaration

## Purpose

One question: *what may this text be?* The four chooser-restricted text adapters answer it; the flat-list converter ([`property_converter_string_values.cpp`](property_converter_string_values.cpp.md)) and the tree chooser ([`property_editor_tree_values.cpp`](property_editor_tree_values.cpp.md)) ask it. Separating it from the adapters is what lets one converter and one chooser serve both binding flavours and both list flavours.

A substantive interface: what it demands of an implementor is the contract a rebuild must satisfy.

## State

Stateless.

## `values`

**Contract** — Returns the set of admissible values as an ordered sequence of text. Unlike the whole-number label list, position carries no meaning here — the document stores the text itself, so the order is presentation only and may be changed freely. May be recomputed per call; callers must not cache it across a user interaction.

```text
FUNCTION values() -> list<text>
```

**Notes** — The sequence is handed over as a first-in-first-out queue rather than an indexable list, which is the shape's only real statement: consumers are expected to walk it once, in order, and not to address it by position. Both consumers do exactly that — one builds a flat choice list, the other splits each entry on path separators to build a tree.
