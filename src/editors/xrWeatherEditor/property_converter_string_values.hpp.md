# src/editors/xrWeatherEditor/property_converter_string_values.hpp

> Declares the converter that turns a text row's admissible set into a drop-down, and the variant that forbids typing.

**Needs** — [`property_string_values_value_base.hpp`](property_string_values_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md) · [`property_holder_string.cpp`](property_holder_string.cpp.md)
**Tier floor** — T3: pure presentation; it never touches engine memory

## Purpose

Declares the converter implemented in [`property_converter_string_values.cpp`](property_converter_string_values.cpp.md), and defines outright a second converter that exists only to answer one question differently.

## Exported units

- **`property_converter_string_values`** — publishes a text row's admissible set to the grid as the row's standard values, and declares that set exclusive. Contract in the implementation.
- **`property_converter_string_values_no_enter`** — the same, with typing refused.

## `property_converter_string_values_no_enter`

**Contract** — Behaves exactly as its base except that it reports no ability to convert *from* text, for any source shape. The grid has no other path by which a typed string could become a value, so the drop-down becomes the only way to change the row.

**Notes** — Refusing a conversion is how "this field is a choice, not a free-text box" is stated to a grid that has no such concept. It is worth naming as a pattern because the same trick appears twice more in this directory — [`property_converter_tree_values.cpp`](property_converter_tree_values.cpp.md) is nothing but this refusal, applied to rows whose chooser is a modal window instead of a drop-down. A rebuild whose property grid can be told "no free text" directly should say so directly and delete all three.
