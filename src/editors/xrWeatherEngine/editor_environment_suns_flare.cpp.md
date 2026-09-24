# src/editors/xrWeatherEngine/editor_environment_suns_flare.cpp

> One flare in a sun's series: where on the line it sits, how big, how bright, and what it looks like.

**Needs** — [`editor_environment_suns_flare.hpp`](editor_environment_suns_flare.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`ide.hpp`](ide.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`editor_environment_suns_flare.hpp`](editor_environment_suns_flare.hpp.md)
**Tier floor** — T3: four values and a file browser.

## Purpose

The element type of the flare series. It has no load or save of its own: the series
decodes the four parallel lists into these records and would have to re-encode them; see
[`editor_environment_suns_flares.cpp`](editor_environment_suns_flares.cpp.md).

Unreachable in the shipping editor.

## State

See [`editor_environment_suns_flare.hpp`](editor_environment_suns_flare.hpp.md).

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder("flare", collection, owner = self)
  add "texture"  (file browser, .dds, start folder = the texture folder, extension dropped)
  add "opacity"  (real, bound to the field)
  add "position" (real, bound to the field)
  add "radius"   (real, bound to the field)
```

**Contract** — four rows, all in a group named `flare`, none bounded.

**Notes** — Every flare is labelled `flare` in the grid's holder name rather than by its
position; the *list* labels its entries by position instead. The three numeric rows carry
descriptions that call them gradient rows, copied from the
[gradient](editor_environment_suns_gradient.cpp.md) file next door. Cosmetic.

Nothing here is bounded, although opacity is plainly a zero-to-one quantity and the rest of
this module bounds such fields. Unreachable code, unreviewed.
