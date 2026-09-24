# src/editors/xrWeatherEditor/property_converter_boolean_values.cpp

> Turns a boolean into one of two author-supplied words, and offers exactly those two words as the dropdown.

**Needs** — [`property_converter_boolean_values.hpp`](property_converter_boolean_values.hpp.md) · [`property_boolean_values_value.hpp`](property_boolean_values_value.hpp.md) · [`property_container.hpp`](property_container.hpp.md)
**Used by** — [`property_boolean_values_value.cpp`](property_boolean_values_value.cpp.md) · [`property_boolean_values_value_reference.cpp`](property_boolean_values_value_reference.cpp.md) · [`property_converter_boolean_values.hpp`](property_converter_boolean_values.hpp.md)
**Tier floor** — T3: a lookup and a render.

## Purpose

The presentation half of the two-label boolean. [`property_boolean_values_value`](property_boolean_values_value.cpp.md) holds the boolean and the two labels and parses a chosen label back; this renders the boolean as a label and supplies the dropdown's contents.

It is the first of four converters in this directory with exactly the same shape, and the shape is worth naming once: **the converter reaches the binding through the row being rendered, reads the choice list off it, and maps between stored value and displayed label.**

## State

`Stateless.` Everything it needs is read off the binding each time.

## The routing

```text
FUNCTION binding_for(context) -> ChoiceBinding
  container  = the object whose row is being rendered
  descriptor = the row
  RETURN container.property_for(descriptor.spec)
```

**Notes** — the same walk [`PropertyGrid`](../xrSdkControls/Controls/PropertyGrid.cs.md) and [`property_collection_editor`](property_collection_editor.cpp.md) make, for the same reason: the grid hands out a row, and the binding is what actually holds the data.

## The dropdown

**Contract** — the offered values are exactly the binding's two labels, and the list is **exclusive**: the author may only choose one of them, not type a third.

**Invariants** — exclusivity is what makes [the binding's parse](property_boolean_values_value.cpp.md) safe, since it asserts that the chosen label is one of the two.

## Rendering

**Contract** — a boolean renders as `labels[1]` when true and `labels[0]` when false. Rendering is skipped — deferred to the default — in four cases: no context, no object, no row, or a destination that is not text. A value that is already text passes through unchanged.

```text
FUNCTION to_text(context, value) -> text
  IF context, object or row is absent THEN default
  IF the destination is not text THEN default
  IF value is already text THEN RETURN value       # the author's own choice, echoed back
  RETURN binding_for(context).labels[value ? 1 : 0]
```

**Notes** — the four guards are not defensive noise. The grid renders values in contexts where no row is selected — a tooltip, a size measurement — and a converter that assumed a row would fault there. A rebuild whose renderer is always given its row deletes them.

The "already text" case is the author's just-committed choice coming back through the renderer before the binding has been re-read. Passing it through avoids a round trip that would yield the same string.
