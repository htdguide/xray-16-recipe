# src/editors/xrWeatherEditor/property_editor_color.cpp

> Paints a colour row's swatch and opens a picker for it, converting between the engine's unbounded reals and the picker's bytes at both ends.

**Needs** — [`property_editor_color.hpp`](property_editor_color.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: pure presentation; the colour crosses by value through the row's own interface

## Purpose

The two affordances a colour property gets beyond its three component rows: a filled rectangle in the row so the artist can scan a keyframe's palette at a glance, and a modal picker.

## State

Stateless — one instance serves every colour row; the row it acts for arrives in each call's context.

## `PaintValue`

**Contract** — Fills the supplied rectangle with the row's current colour. Reads the colour through the row's own interface, never from the painted value handed in. Does nothing when there is no value.

```text
FUNCTION PaintValue(surface, bounds, subject)
  colour = subject.owner.get_value_raw()       # the engine's current value
  fill(surface, bounds, to_byte_colour(colour))

FUNCTION to_byte_colour(c) -> (int, int, int)
  # clamp above, round to nearest, no clamp below
  RETURN (round(min(c.r, 1) * 255), round(min(c.g, 1) * 255), round(min(c.b, 1) * 255))
```

**Notes** — Three decisions sit in that one conversion.

*Clamp at unity, not below.* The engine's colour channels are unbounded reals and legitimately exceed one — a sun colour is authored bright. The swatch must show something, so it saturates. Nothing clamps at zero, because nothing in the authored data goes negative; a rebuild that cannot assume that must clamp both ends.

*Round to nearest, not truncate.* Truncating would make a channel authored as exactly one display one step below full. The rounding is done by adding a half before converting.

*Read through the row, not from the value passed in.* The value the grid offers for painting is a container, not a colour — a colour row's value *is* its expandable child set — so the actual colour has to be fetched from the adapter behind it. This is the same subject-to-container-to-adapter walk every presentation file here performs.

The brush is released as soon as the fill is done rather than left to the collector, because painting happens on every repaint of every visible row and the drawing resource behind it is scarce.

## `GetPaintValueSupported`

**Contract** — Always true.

## `GetEditStyle`

**Contract** — Modal whenever there is a row context; otherwise defers to the grid's default. The editor is being asked to describe itself outside any particular row in that case, and has nothing to say.

## `EditValue`

**Contract** — Opens the platform colour picker seeded with the row's current colour, fully expanded so the custom-colour area is available without a further click. On anything but cancel, writes the picked colour back through the row's adapter. Blocks until the dialog closes. Returns the value it was handed, unchanged — the write has already happened through the adapter.

```text
FUNCTION EditValue(context, services)
  IF no context OR no services OR no window service THEN RETURN default

  adapter = context.subject.property(context.row)    # AS colour row
  colour  = adapter.get_value_raw()

  dialog = colour_picker(seed: to_byte_colour_truncating(colour), fully_open: true)
  IF dialog.show() != cancelled THEN
    adapter.SetValue(from_byte_colour(dialog.colour))   # each channel / 255
  RETURN default
```

**Invariants** — A colour that leaves and re-enters the picker unchanged must come back equal. It does not, for any channel above one: seeding truncates rather than clamping, so a channel at `1.4` seeds as an overflowed byte and comes back as something unrelated. This is the sharpest edge in the file — a high-range colour opened and confirmed without touching anything is silently altered.

**Notes** — The picker is the boundary between an authoring model that allows overbright colour and a platform control that does not. The editor accepts the loss rather than replacing the control, which is a defensible trade for a tool but needs stating, because the weather keyframes it edits are exactly where overbright sun and sky colours live. A rebuild has three options: clamp on seeding so at least the round trip is idempotent, refuse the picker for out-of-range colours, or supply its own picker that speaks reals.

Nothing happens unless the grid supplies its modal-hosting service — the editor gives up and defers rather than opening an unowned window, which would sit behind the tool and take the keyboard with it.
