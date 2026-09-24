# src/xrUICore/Buttons/UIRadioButton.cpp

> Pins the radio graphic, offsets the label past it, and re-announces the tab group's change as a radio selection.

**Needs** — [`UIRadioButton.h`](UIRadioButton.h.md) · [`TabControl/UITabButton.h`](../TabControl/UITabButton.h.md) · [`Lines/UILines.h`](../Lines/UILines.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UIRadioButton.h`](UIRadioButton.h.md)
**Tier floor** — T3.

## Purpose

A very thin file, and honestly so: the interesting decision was made in the header by
choosing the base class. What remains is the fixed appearance and one message rename.

## `init_button`

**Contract** — runs the tab button's own initialization, forces the texture set named
`ui_radio` (the same `_e`/`_d`/`_t`/`_h` convention as any four-state button), then lays the
label out: the text is offset right by the width of the dot graphic, the button's own height
is reduced to the dot's height less five units, and the label's box is set to the full
requested width by the dot's full height.

**Notes** — the five-unit shrink is unexplained by anything discoverable in the source. It
makes the clickable area slightly shorter than the graphic; the most plausible reading is
that it stops vertically adjacent radio buttons in a shipped layout from overlapping their
hit rectangles. Recorded as unrecovered.

The label box is deliberately taller than the button: the text control's vertical centring
then centres against the graphic rather than against the shortened hit area.

## `init_texture`

**Contract** — accepts any texture name and does nothing, reporting success. The radio
button's appearance is not data-driven; an XML element that names a texture for a radio
button is silently ignored. This must be preserved — the shipped XML does name textures there.

## `send_message` / `on_mouse_down`

**Contract** — both funnel into one behaviour: whenever this button becomes the active tab —
either because the tab control announced the change naming this button, or because this
button was just pressed — announce `RADIOBUTTON_SET` to the message target. A disabled radio
button swallows every message and announces nothing.

**Notes** — there is no matching "unset". A listener learns only who is now selected, which
is sufficient because the group guarantees exactly one. Both paths can fire for a single
user action (press, then the tab control's change notification), so a listener must tolerate
a duplicate set for the same button.
