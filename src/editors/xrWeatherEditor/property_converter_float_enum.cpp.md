# src/editors/xrWeatherEditor/property_converter_float_enum.cpp

> Renders a real as the label of the nearest declared choice — an enumeration whose underlying values happen to be real numbers.

**Needs** — [`property_converter_float_enum.hpp`](property_converter_float_enum.hpp.md) · [`property_float_enum_value.hpp`](property_float_enum_value.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_converter_float_enum.hpp`](property_converter_float_enum.hpp.md)
**Tier floor** — T3: a linear search and a render.

## Purpose

Some weather fields are a real number chosen from a small named set — a blend curve, a flare shape factor. The value the engine stores is a real; what the author should see and pick is a name.

Same shape as [`property_converter_boolean_values`](property_converter_boolean_values.cpp.md), over a list of (value, label) pairs rather than two positional labels.

## State

`Stateless.` The pair list is read off the binding each time.

## The dropdown

**Contract** — the offered values are the binding's (value, label) pairs, exclusive.

**Notes** — the pairs are offered as *pairs*, not as labels, which is why the renderer below has to handle being given a pair. The grid renders whatever it was offered; the author's selection comes back as the pair they picked.

## Rendering

**Contract** — renders three kinds of input, in order: text passes through; a pair renders as its label; a bare real is matched against the pairs and renders as the matching label, falling back to the **first** pair's label when nothing matches.

```text
FUNCTION to_text(context, value) -> text
  IF context, object or row is absent THEN default
  IF the destination is not text THEN default
  IF value is already text THEN RETURN value
  IF value is a (value, label) pair THEN RETURN its label
  FOR EACH pair IN binding_for(context).pairs
    IF pair.value == value THEN RETURN pair.label
  RETURN the first pair's label                   # no match: show the first choice
```

**Notes** — the fallback is the file's one real decision and it is a lie: a stored value outside the declared set is displayed as the first choice, so the author sees a value the data does not hold, and a single touch of that row commits the lie.

It is chosen over showing the raw number because the row's cell is a dropdown and a raw number is not one of its items. The honest alternative — show the number, mark the row as out-of-set — needs a cell editor that can do both. A rebuild should do that, or should at least fall back to rendering the number.

The match is an **exact equality test on a real**. It works because both sides are values the engine wrote from the same declared list, never computed ones. It would not survive a value that had been through arithmetic, and a rebuild that stores these as an index or a name rather than as a real removes the hazard entirely — which is what an enumeration should have been in the first place.
