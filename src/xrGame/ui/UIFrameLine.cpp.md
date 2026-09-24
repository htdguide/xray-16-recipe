# src/xrGame/ui/UIFrameLine.cpp

> A stretchable one-axis frame drawn directly as three sprites — a start cap, an end cap and a tiled middle — with a widescreen width correction baked into the end cap. Excluded from the build.

**Needs** — [`UIFrameLine.h`](UIFrameLine.h.md) · [`UIStaticItem.h`](../../xrUICore/Static/UIStaticItem.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`HUDManager.h`](../HUDManager.h.md)
**Used by** — [`UIFrameLine.h`](UIFrameLine.h.md)
**Tier floor** — T2: it tiles a texture by computing an integer repeat count and a fractional remainder, and submits sprites rather than widgets.

## Purpose

The ancestor of chapter 15's frame line, kept in the game layer for elements drawn outside
the widget tree — heads-up decoration composited with the world rather than with the screen.
It is **commented out of the build**; the surviving frame line is the toolkit's. It is
described because it carries one decision the toolkit's version states differently and one
the toolkit does not state at all.

## State

```text
RECORD FrameLine
  start_cap, end_cap, middle : Sprite
  origin        : point
  length        : real          # along the line's axis
  horizontal    : bool
  stretch       : bool          # thicken to the owner's cross-axis size
  owner_size    : (real, real)  # the owner's box, needed only when stretching
  size_is_valid : bool          # geometry cache flag
```

**Invariants**

- The two caps must be the **same thickness** on the cross axis; this is asserted, because
  a mismatch produces a line with a step in it that is very hard to see in a texture and
  obvious on screen.
- The line's length must exceed the two caps' combined length; a shorter line would give
  the middle a negative width. Also asserted.
- Geometry is recomputed lazily: changing origin, length or orientation invalidates, and the
  next draw recomputes. Nothing recomputes eagerly, because a screen typically sets all three
  in sequence.

## Texture naming

**Contract** — one texture name yields three sprites by suffix: `_b` is the start cap, `_e`
is the end cap, `_back` is the middle.

**Notes** — the suffixes are a **frozen data convention**: they are not written anywhere in
the shipped layouts, which name only the stem. A rebuild must keep these three exact
suffixes or every shipped frame line fails to resolve its textures.

## Laying out the three pieces

```text
FUNCTION recompute()
  place start_cap at origin
  cap_len := end_cap's natural length along the axis
  IF horizontal AND the display is widescreen THEN cap_len := cap_len / 1.2
  place end_cap at the far end, inset by cap_len
  middle_span := length - start_cap length - cap_len
  repeats   := floor(middle_span / middle's natural length)
  remainder := middle_span modulo middle's natural length
  place middle after the start cap, tiled `repeats` times plus `remainder` units
  mark geometry valid
```

**Notes** — the tiling is *whole repeats plus a partial one*, not a stretch. A frame line's
middle texture usually carries a pattern — rivets, a gradient ribbing — and stretching it
would change its pitch with the widget's width. The partial repeat at the end is accepted as
a cut-off motif. This is the same choice chapter 15's frame line offers as an option; here it
is the only behaviour.

**The 1.2 correction is the interesting number.** Chapter 15 states that the canvas is scaled
*non-uniformly* to the back buffer, so a wide display stretches everything horizontally, and
that the engine corrects for it in exactly two places. This is one of them, in its raw form:
on a 16:9 display the end cap of a horizontal line is narrowed by a factor of 1.2 in canvas
units so that after the horizontal stretch it comes out at its authored proportions. 1.2 is
the ratio of the two aspect ratios the engine distinguishes — 16:9 over 4:3 — and it is
applied only to the *end* cap and only when horizontal. The start cap is not corrected, so a
16:9 line is not symmetric: its right end is visibly narrower than its left. That asymmetry
is in the shipped look and a rebuild that "fixes" it changes the game's appearance.

The correction is applied twice — once when positioning the end cap and once when sizing it —
and the two must agree or the cap and its slot disagree.

## Stretching across the axis

**Contract** — when the stretch flag is set, every piece is drawn at the *owner's* cross-axis
size rather than at its texture's natural thickness: a horizontal line takes the owner's
height, a vertical line the owner's width. Along the axis nothing changes.

**Notes** — this is how one thin authored texture dresses a row of any height. The owner's
size must have been supplied first, and an unset owner size is a fault rather than a default,
because zero would make the line invisible with no diagnostic.

## `Render`

**Contract** — recompute if stale, then size and submit the three sprites in order: start cap,
end cap, middle. The middle is drawn last and therefore **over** the caps where they meet,
which is why cap textures are authored with their seam edge opaque.
