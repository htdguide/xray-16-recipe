# src/editors/xrWeatherEditor/property_converter_integer_values.cpp

> Renders an integer as a label *indexed by* the integer — the case where the stored value is the position in the choice list.

**Needs** — [`property_converter_integer_values.hpp`](property_converter_integer_values.hpp.md) · [`property_integer_values_value_base.hpp`](property_integer_values_value_base.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_converter_integer_values.hpp`](property_converter_integer_values.hpp.md)
**Tier floor** — T3: an index and a render.

## Purpose

The third of the three integer-choice shapes, and the one that is genuinely different from the other two. Here the engine supplies a plain list of labels and the stored integer **is the index into it** — no pairs, no search.

It exists alongside [`property_converter_integer_enum`](property_converter_integer_enum.cpp.md) because some engine enumerations really are dense zero-based sequences and declaring the values explicitly would be noise.

## State

`Stateless.`

## The dropdown

**Contract** — the binding's label list, offered exclusively.

## Rendering

**Contract** — text passes through; anything else is treated as an integer and used to index the label list directly.

```text
FUNCTION to_text(context, value) -> text
  IF context, object or row is absent THEN default
  IF the destination is not text THEN default
  IF value is already text THEN RETURN value
  RETURN binding_for(context).labels[value]       # unchecked
```

**Notes** — **the index is not bounds-checked.** Where the enumeration converters fall back to the first label for an out-of-set value, this one indexes past the end and faults. A keyframe written by a build with more choices than the running one crashes the editor rather than mis-rendering.

Nothing about the design requires that: the fix is one comparison. A rebuild bounds-checks, and should prefer rendering the raw number over any fallback label — for the reasons given in [`property_converter_integer_enum`](property_converter_integer_enum.cpp.md).

The paired binding class is reached as its *base* type here, not as one of the four concrete forms, which is the one place in this directory where the label-list interface is abstracted rather than duplicated. It is the pattern the rest of the directory should have followed.
