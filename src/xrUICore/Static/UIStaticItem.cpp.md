# src/xrUICore/Static/UIStaticItem.cpp

> The one place in the chapter where a widget becomes triangles — it turns a rectangle, a sub-rectangle of an atlas page and a colour into a clipped triangle fan in the renderer's stream, with or without rotation.

**Needs** — [`UIStaticItem.h`](UIStaticItem.h.md) · [`ui_defs.h`](../ui_defs.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIStaticItem.h`](UIStaticItem.h.md)
**Tier floor** — T1: it emits vertices straight into the renderer's primitive stream, and
runs once per drawn widget per frame.

## Purpose

Every visible thing in the interface — a picture, a button face, a frame corner, a list row's
background — eventually becomes this: four corners, four texture coordinates, one colour,
clipped and emitted. Concentrating that here is what makes the widget layer's several dozen
control types share one rasterisation path and one clipping policy.

Two decisions inside it shape how the shipped art looks.

**Texture coordinates are normalised at draw time**, from the page's *actual* resolution as
the device reports it. The registry stores texel rectangles because the page may not be
resident when the layout is read; the division happens here, every frame, and it is why the
same atlas description works against a page shipped at two different sizes.

**A half-pixel offset is applied to positions.** It is the texel-centre correction the
original graphics API needed so that an unscaled sprite lands exactly on pixels rather than
straddling them. It is suppressed for world-space UI, where there are no pixels to land on.

## State

```text
RECORD StaticItem
  position       : vec2        # canvas units; set by the owning widget each frame
  size           : vec2        # canvas units
  texture_rect   : Rect        # texels of the page
  colour         : colour      # modulates the sampled texel
  material       : Material    # shared, from the icon registry
  mirror         : { None | Horizontal | Vertical | Both }
  heading_pivot  : vec2        # rotation anchor, in the item's own units
  heading_offset : vec2        # translation applied after rotation
  flags          : { size_is_valid, rect_is_valid, pivot_is_valid, pin_top_left_when_rotating }
```

**Invariants**

- `size` and `texture_rect` each carry a validity flag. If either is unset when the item is
  first drawn, it is **filled in from the material's base texture resolution** — a picture
  with nothing said about it draws at its page's full size. Creating a new material clears
  both flags, so a re-textured item re-derives them.
- The pivot's validity flag distinguishes "rotate about the authored pivot" from "rotate
  about the centre". The centre is the default and is computed, not stored.
- The item does not know about the widget tree. It draws at the position it is given, and the
  owner computes that from the absolute rectangle.

## `Render` (unrotated)

**Contract** — binds the material, opens a primitive batch sized for the worst case, emits
the clipped quad, and flushes. Asserts it is inside the frame's render bracket.

```text
FUNCTION render(item)
  bind item.material
  open a triangle-list batch with room for 8 vertices
  render_internal(item, item.position)
  flush
```

**Notes** — the batch is opened and flushed per item, so there is no batching *across*
widgets. That is a real cost and a deliberate simplification: the draw order is the widget
tree's order, and reordering to batch would change which widget draws on top. The text layer
takes the opposite trade and batches everything (see
[`ui_base.cpp`](../ui_base.cpp.md)).

## `RenderInternal(position)` — the unrotated path

**Contract** — builds the quad in screen pixels, normalises its texture coordinates, applies
mirroring, clips against the current screen frustum, and emits the survivor as a fan.

```text
FUNCTION render_internal(item, pos)
  p = to_screen(pos); align p to whole pixels
  page = the material's base texture resolution
  IF item.size is unset      THEN item.size = page
  IF item.texture_rect unset THEN item.texture_rect = the whole page

  top_left     = p
  bottom_right = p + to_screen(item.size)

  uv_top_left     = item.texture_rect.top_left     / page      # normalise here
  uv_bottom_right = item.texture_rect.bottom_right / page

  IF mirroring horizontally THEN swap the two u components
  IF mirroring vertically   THEN swap the two v components

  offset = -0.5 pixels, or 0 when drawing world-space UI
  shift both corners by `offset`

  quad = the four corners with their texture coordinates, clockwise from top-left
  clipped = clip quad against the current frustum (screen or world-space)
  IF clipped is non-empty
    emit clipped as a triangle fan anchored at vertex 0, all vertices coloured item.colour
```

**Invariants**

- Only the *position* is pixel-aligned, and only the position receives the half-pixel offset.
  Texture coordinates are left exact, so an aligned sprite samples texel centres.
- Mirroring is done by swapping texture coordinates, not by flipping geometry, so winding
  order is unaffected.
- The frustum used depends on the current point type: the screen frustum for normal UI, the
  world-space frustum for UI drawn onto a surface in the world.

## `RenderInternal(angle)` — the rotated path

**Contract** — the same quad, but built in the item's local space, rotated about the pivot,
translated to the item's position, and only then converted to screen coordinates.

```text
FUNCTION render_internal_rotated(item, angle)
  page = the material's base texture resolution
  half_texel = 0.5 / page                       # half-texel bias, see notes
  IF item.size is unset      THEN item.size = page
  IF item.texture_rect unset THEN item.texture_rect = the whole page

  pivot  = item.heading_pivot IF valid ELSE the item's centre
  origin = item.position + item.heading_offset
  uv     = item.texture_rect / page, each corner biased by +half_texel
  apply mirroring by swapping u and/or v as above

  aspect = the horizontal canvas stretch factor
  FOR EACH corner of the rectangle (0,0)-(size.x, size.y)
    rotate it about `pivot` by `angle`, scaling x by `aspect`
    translate it by `origin`
    convert it to screen coordinates

  clip against the screen frustum and emit as a fan
```

**Invariants**

- The aspect factor is applied *inside* the rotation, so a rotated sprite stays
  proportionate on a wide display even though the canvas is stretched non-uniformly.
- No pixel alignment and no half-pixel position offset in this path: a rotated sprite has no
  axis-aligned edges to snap, and snapping would make it wobble as the angle changes.
- The rotated path always uses the screen frustum, never the world-space one. Rotated
  world-space UI would be clipped wrongly; nothing in the shipped data does it.

**Notes** — the half-texel bias added to every texture coordinate here, and absent in the
unrotated path, is the counterpart to the half-pixel position offset that the unrotated path
applies and this one does not. Together the two paths are consistent about where a texel
centre lands; separately, neither is obviously right. A rebuild targeting an API with
texel-centre-at-pixel-centre sampling drops both corrections, and must then verify the
unscaled art still looks sharp.

## `CreateShader` / `Init` / `SetShader`

**Contract** — `CreateShader` builds a material from a texture path and a pass name and
invalidates the derived size and rectangle, so they are re-derived from the new page.
`Init` does that and sets a position. `SetShader` installs an already-built material and
does **not** invalidate, because the caller is sharing a material whose rectangle it is about
to set itself.

## `SetHeadingPivot` / `ResetHeadingPivot`

**Contract** — set or clear the rotation anchor and the post-rotation offset, plus the flag
saying whether the item's top-left corner stays put while it rotates. Clearing returns to
rotating about the centre.

**Notes** — the pinned-top-left flag is read by the owning widget, not here: it is what makes
[`UIStatic.cpp`](UIStatic.cpp.md) transpose the widget's rectangle before a rotated stretch.
