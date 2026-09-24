# src/xrGame/ui/UIPdaKillMessage.cpp

> One kill report: four optional elements packed left to right, each contributing its width to
> the next one's position, with the whole row fading itself out after five seconds.

**Needs** — [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md)
**Used by** — [`UIPdaKillMessage.h`](UIPdaKillMessage.h.md)
**Tier floor** — T3.

## Purpose

A kill report reads as a sentence made of pictures: *name, weapon icon, name, circumstance
icon*. Any of the four may be missing — a death with no killer, a kill with no special
circumstance — and the row must close up rather than leave gaps. That packing is the file.

## State

```text
RECORD KillReport EXTENDS FadingContainer
  killer_name : Static    # plain text, no inline markup
  initiator   : Static    # the weapon or cause icon
  victim_name : Static    # plain text, no inline markup
  ext_info    : Static    # a circumstance icon: headshot, friendly fire, and so on
```

**Invariants**

- The row's *height* is authored and never changes; every element is vertically centred within
  it. Only the width grows.
- The two name widgets have inline markup **disabled**, because a player's name is arbitrary
  text and a name containing the markup's escape would otherwise be interpreted. The colour
  comes from the record instead.

## `Init`

**Contract** — place the four elements left to right, each after the previous one plus a
three-unit gap, skipping any that the record left empty and charging no gap for it. Then widen
the row to the cursor plus the last icon's width, and start a five-second fade.

```text
FUNCTION init(record, font)
  x <- 0
  FOR EACH (widget, part) IN [(killer_name, record.killer), (initiator, record.initiator),
                              (victim_name, record.victim), (ext_info,  record.ext_info)]
    w <- place(widget, at x, from part)        # 0 when the part is empty
    IF w > 0 AND widget IS NOT the last THEN x <- x + w + 3
  width <- max(width, x + ext_info.width)
  start the named fade over 5 seconds
```

**Notes** — the final element is placed but its width is not added to the cursor; the row's
width is computed from the cursor plus that width instead. The two are the same number, so the
asymmetry is cosmetic.

## `InitText`

**Contract** — for a non-empty coloured name: set the widget's position with its text vertically
centred in the row, give it the row's full height, enable ellipsis, set the text and its colour,
shrink the widget to its text, and then add **one space's width** of trailing room. Report the
resulting width. An empty name reports zero and is not placed.

**Notes** — the extra space is added because shrinking a widget to its text leaves no room for
the glyph's own right bearing, and an adjacent icon would touch the last letter. The space is
measured in *screen* units after a conversion, while the widget's width is in canvas units — so
on a wide display the padding is wider than intended. This is the same unit confusion as in
[`UIOutfitInfo`](UIOutfitInfo.cpp.md) and equally preserved.

## `InitIcon`

**Contract** — for an icon with a non-empty source rectangle whose material is ready: scale it
down (never up) so its height fits the row, centre it vertically, bind it stretched to that
source rectangle and material, and report its scaled width. An absent icon, or one whose
material has not loaded, reports zero and is not placed.

**Notes** — the material-readiness check is what keeps a kill report from drawing a blank box
while a weapon's atlas is still streaming; the icon simply does not appear and the row closes up
around it. A rebuild with synchronous asset loading needs no check but still needs the
close-up behaviour for genuinely absent parts.
