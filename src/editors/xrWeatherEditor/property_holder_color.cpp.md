# src/editors/xrWeatherEditor/property_holder_color.cpp

> Registers colour properties, giving each one a swatch in the grid row and a modal picker.

**Needs** — [`property_holder.hpp`](property_holder.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_color.hpp`](property_color.hpp.md) · [`property_color_reference.hpp`](property_color_reference.hpp.md) · [`property_editor_color.hpp`](property_editor_color.hpp.md) · [`property_converter_color.hpp`](property_converter_color.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: converts between the engine's three-real colour record and the presentation layer's colour

## Purpose

Two registration overloads — colour by callback pair, colour by field alias. Colours are the most-edited quantity in a weather keyframe (sky, fog, ambient, sun, hemisphere), so they get the richest presentation: a painted swatch in the row, a modal picker, and an expandable set of component rows.

## State

Stateless.

## `add_property` (colour)

**Contract** — As in [`property_holder_boolean.cpp`](property_holder_boolean.cpp.md), with one difference: the presentation record is built first and then handed to the adapter, because the adapter needs the record's attribute set to describe its own expandable child rows to the grid.

```text
FUNCTION add_property(identifier, category, description, default, binding)
  spec = PresentationSpec {
    name        = copy(identifier),
    declared    = colour,
    category    = copy(category),
    description = copy(description),
    default     = presentation_colour(default.r, default.g, default.b),
    editor      = colour_picker,
    converter   = colour_converter
  }
  adapter = accessor_bound OR reference_bound colour, given spec.attributes
  container.add_property(spec, adapter)
  RETURN none
```

**Notes** — The engine's colour is three unbounded reals, one per channel; the presentation layer's colour is three bytes. The conversion is lossy in one direction and range-losing in the other, and the editor accepts that: the picker can only express what three bytes can, so a channel the engine holds above unity is not recoverable once the user opens the picker. The swatch painter and the picker each clamp before converting — see [`property_editor_color.cpp`](property_editor_color.cpp.md). A rebuild that wants high-range colour authoring has to replace the picker, not the binding.
