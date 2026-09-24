# src/xrGame/ui/UIWeightBar.cpp

> The carried-weight line: a caption, a weight and a limit, packed right to left so the group stays
> together as the numbers change width.

**Needs** — [`UIWeightBar.h`](UIWeightBar.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md)
**Used by** — [`UIWeightBar.h`](UIWeightBar.h.md)
**Tier floor** — T3: text measurement and horizontal packing

## Purpose

The inventory and trade screens both show "Weight: 23.4 / 60.0 kg" somewhere. The numbers change
width as they change value, and the three labels must stay visually attached — so, exactly as in
[`CUITradeBar`](UITradeBar.cpp.md), the line is packed **right to left** from a fixed anchor rather
than laid out at authored positions.

The one structural difference from the trade line is that its element names are built from a
caller-supplied prefix rather than being fixed, so one screen can hold several of these — the trade
screen has one per side.

## State

```text
RECORD WeightLine extends Window
  caption    : Widget                  # "Weight:"
  weight     : Widget                  # the current load
  weight_max : optional<Widget>        # the limit
  anchor_x   : real                    # where packing starts from
  bag, bag_2 : optional<Widget>        # carry-capacity indicators, supplied by the owner
```

Invariants:

- `anchor_x` is captured at build time from the **limit** label's position when there is one, and
  from the weight label's position when there is not. Everything is then placed leftward from it, so
  the line's right edge is stable whatever the numbers are.
- The two indicators are not built here and not owned here; the owning screen assigns them, and both
  updates tolerate their absence.

## `init_from_xml`

**Contract** — Builds three labels whose element names are the caller's prefix with the fixed
suffixes `_weight_caption`, `_weight` and `_weight_max`, the last optional. Captures the packing
anchor and shrinks the caption to its text.

## `UpdateData(weight)`

**Contract** — The bare form: a weight with the localized kilogram abbreviation, no limit. Shrinks
the weight label and the caption to their text and packs both leftward from the anchor. Does nothing
if either label is missing.

```text
FUNCTION UpdateData(weight)
  IF weight label OR caption IS missing THEN RETURN
  weight.text <- weight to one decimal + " " + localized("st_kg")
  weight.fit_width_to_text(); caption.fit_width_to_text()
  weight.x  <- anchor_x - weight.width - 5
  caption.x <- weight.x - caption.width - 5
```

## `UpdateData(owner)`

**Contract** — The full form: updates the two optional carry-capacity indicators from the owner —
one with the limit included and one without, which is how the inventory shows both the bag's
capacity and the player's total — then fills the weight and limit labels from the owner through the
shared inventory formatting helper, shrinks all three to their text, and packs the caption and the
weight leftward from the **limit label's current position** rather than from the captured anchor.

```text
FUNCTION UpdateData(owner)
  IF bag   EXISTS THEN update it from owner, including the limit
  IF bag_2 EXISTS THEN update it from owner, excluding the limit
  IF weight OR weight_max OR caption IS missing THEN RETURN
  fill weight and weight_max from owner through the shared helper
  fit all three to their text
  weight.x  <- weight_max.x - weight.width - 5
  caption.x <- weight.x - caption.width - 5
```

**Notes** — The two overloads pack against different anchors — the captured one and the live limit
label — which are the same value unless something moved the limit label. Nothing does in the shipped
screens, so the difference is latent; a rebuild should pick one.

The 5-unit gaps are authored spacing, the same as the trade line's. Both overloads recompute
positions from scratch each call rather than tracking a layout, which is what makes them safe to
call every frame.
