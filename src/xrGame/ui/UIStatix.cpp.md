# src/xrGame/ui/UIStatix.cpp

> A picture that acts as a choice: it sends a click notification like a button, tints green under
> the cursor, dims when disabled, and pulses while it is the selected one.

**Needs** — [`UIStatix.h`](UIStatix.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIStatix.h`](UIStatix.h.md)
**Tier floor** — T3: colour state over a picture widget

## Purpose

The team and skin pickers present their choices as large pictures rather than labelled buttons.
Rather than make the button widget able to carry a full-size picture, the engine takes the other
route: teach the picture the three things a choice needs — to report a click, to highlight under the
cursor, and to show that it is the current selection.

## State

```text
RECORD SelectablePicture extends Picture
  selected : bool     # drives the pulse animation, nothing else
```

Invariants:

- The pulse is started and stopped only on a *change* of `selected`; setting it to the value it
  already has does nothing, so repeated assignment does not restart the animation mid-cycle.
- The tint is recomputed from scratch every frame rather than tracked, so no state can drift.

## The tint rule

**Contract** — Recomputed every frame, in this order, last wins:

1. base: the picture is fully opaque white (untinted);
2. if the cursor is over the widget: a green tint;
3. if the widget is disabled: white at half alpha.

So a disabled choice does not highlight, and a highlighted choice is never dimmed.

**The redirection through a named child** is the part worth keeping: if the widget has a child named
`auto_static_0` — the name the layout reader gives the first anonymous decorative child it builds —
the tint is applied to *that* child and this widget is left untinted. The layouts dress a choice as
a plain picture with a separate frame overlay, and tinting the frame rather than the photograph is
what makes the highlight read. When there is no such child, the widget tints itself.

```text
FUNCTION Update()
  frame <- child named "auto_static_0", if any
  IF frame EXISTS THEN frame.tint <- transparent ELSE (nothing)
  self.tint <- opaque white

  IF cursor is over this widget THEN
    IF frame EXISTS THEN frame.tint <- green ELSE self.tint <- green

  IF NOT enabled THEN self.tint <- white at half alpha
  base.Update()
```

**Notes** — The green is a specific authored colour, not a named theme entry, and the disabled state
is half alpha on white. Neither is configurable; both are frozen by how the shipped pickers look.
The frame child's *untinted* state is fully transparent, not white — an unhighlighted frame is
invisible, and the highlight is the frame appearing rather than changing colour.

## `OnMouseDown`

**Contract** — Any press anywhere on the picture sends a **click** notification to the widget's
message target and consumes the event. There is no press/release state machine and no hit region
smaller than the widget: the picture is one big click target. The notification id is the same one a
button sends, which is why the pickers tell their pictures and their buttons apart by sender rather
than by message.

## `SetSelectedState`

**Contract** — On a change to selected, starts a slow cyclic pulse affecting only the alpha of the
text and texture colours. On a change to unselected, first runs the focus-lost path — which resets
the tint to its resting value — and then stops the pulse. Doing it in that order matters: stopping
the animation alone would leave the widget frozen at whatever alpha the pulse had reached.

**Notes** — The pulse is a named colour animation supplied by data (`ui_slow_blinking`), so its
period and shape are authored, not coded. A rebuild needs an equivalent named-animation facility or
must inline the curve.

## `OnFocusReceive` / `OnFocusLost`

**Contract** — Receiving focus restarts the pulse from its beginning, so a selected picture the
player navigates to reads as freshly selected. Losing focus resets the tint to opaque white, or
clears the frame child's tint when there is one, and re-applies the disabled dimming — which must be
re-applied here because this path does not go through the per-frame tint rule.
