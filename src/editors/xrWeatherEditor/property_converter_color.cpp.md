# src/editors/xrWeatherEditor/property_converter_color.cpp

> A colour's one-line form: three space-separated reals that the author can read, type and paste.

**Needs** — [`property_converter_color.hpp`](property_converter_color.hpp.md) · [`property_color.hpp`](property_color.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_converter_float.hpp`](property_converter_float.hpp.md)
**Used by** — [`property_converter_color.hpp`](property_converter_color.hpp.md)
**Tier floor** — T3: string formatting and a fixed row order.

## Purpose

[`property_color_base`](property_color_base.cpp.md) exposes a colour as a container of three rows. That makes the colour expandable but leaves its own row with nothing to show. This supplies the one-line form, in both directions.

The format is deliberately the same one the engine's configuration files use for a colour — three reals separated by spaces — so an author can copy a colour out of the grid, into a text file, and back.

## State

`Stateless.`

## Row ordering

**Contract** — the three sub-rows are sorted into red, green, blue order and the count is asserted to be exactly three.

**Notes** — without this the grid sorts them alphabetically: blue, green, red. Colour channels have a conventional order that is not alphabetical, and this is where it is imposed. The assertion is the file's one integrity check and it is debug-only.

## Rendering

**Contract** — reads the colour through its container's owner (the composite binding) and renders the three channels with [the real formatter](property_converter_float.cpp.md), joined by single spaces. The colour also converts to the boundary-safe three-real record, which is what the grid hands to a cell editor.

```text
FUNCTION to_text(container) -> text
  colour = container.owner.read_color()
  RETURN format(colour.r) + " " + format(colour.g) + " " + format(colour.b)
```

**Notes** — reaching the value by asking the container for its owner and narrowing is what [`property_container_holder`](property_container_holder.hpp.md) exists for. It is the only place in the editor where a nested value's rows find their way back to the value.

Because it goes through [the real formatter](property_converter_float.cpp.md), the colour inherits that file's three-decimal precision and its locale asymmetry.

## Parsing

**Contract** — splits the text on its first two spaces and parses three reals. Any failure — a missing separator, an unparseable field — rejects the whole edit with the offending text named; a partial colour is never written.

**Invariants** — exactly three fields, separated by exactly the first two spaces found. The third field is whatever remains, so trailing text after the third number is fed to the parser and fails there rather than being ignored.

**Notes** — splitting on the first two spaces rather than on whitespace runs means `0.5  0.5 0.5`, with a doubled space, fails to parse. It is brittle, and a rebuild should split on runs of whitespace. What must be preserved is the all-or-nothing rule: a colour is three numbers or it is a rejected edit.

There is no range clamp. A pasted colour with a channel above one is stored as written, and the running weather will use it — which for a high-dynamic-range ambient term may be exactly what the author wanted, so the absence of a clamp here (where the *component* rows do clamp) is defensible. It is nowhere stated, which is why it is worth stating.
