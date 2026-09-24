# src/xrUICore/Windows/UIFrameWindow.cpp

> Draws a nine-piece frame — four fixed corners, four tiled edges and a tiled interior — from one texture, and refuses to be sized smaller than its own corners.

**Needs** — [`UIFrameWindow.h`](UIFrameWindow.h.md) · [`UIWindow.h`](UIWindow.h.md) · [`UIFrameLineWnd.h`](UIFrameLineWnd.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIFrameWindow.h`](UIFrameWindow.h.md)
**Tier floor** — T1: builds an explicit triangle list with per-vertex texture coordinates and colours and submits it to the device in one primitive.

## Purpose

The panel behind every dialog, inventory pane and message box. Where
[the frame line](UIFrameLineWnd.cpp.md) stretches in one dimension, this stretches in two:
nine pieces loaded from one base name, the four corners drawn once each, the four edges tiled
along their axis, and the interior tiled in both. Every piece keeps its authored size, so the
art's border thickness is constant however large the panel is.

Unlike the line, this widget has only *one* shader for all nine pieces — they must come from
one atlas — so the entire frame is always a single primitive.

## State

```text
RECORD FrameWindow EXTENDS Window, TextureOwner
  shader         : Shader       # one, shared by all nine
  piece_rect     : Rect x 9     # interior, four edges, four corners: source rects in the atlas
  texture_colour : colour       # opaque white
  texture_shown  : bool
  title          : optional<Static>
```

**Invariants**

- `texture_shown` is true only when all nine pieces loaded; a partial set draws nothing.
- The corners and edges must agree on the dimensions they share — top-left with top and with
  top-right on height, top-left with left and with bottom-left on width, and so on. Nine
  consistency rules are checked; the two involving the interior are *not*, because the
  interior tiles freely.
- The widget's size is never smaller than the sum of its corner pieces. This is enforced in
  the size setter, not checked at draw.

## `SetWndSize`

**Contract** — clamps the requested size up to the frame's minimum — the two top corners'
widths and the left corners' heights — before applying it. Only clamps when a texture is
loaded, so an untextured frame window is freely sizable.

**Notes** — this is the one place a widget in this toolkit refuses its caller's geometry, and
it exists because the drawer asserts the interior's extents are non-negative. Without the
clamp a too-small panel would fail rather than look wrong. A rebuild that drops the assert can
drop the clamp, at the cost of overlapping corners.

The clamp round-trips the size through the screen-pixel conversion to compare it against the
source rectangles, which are in atlas pixels — the comparison has to happen in one space and
the art's space is the one that is fixed.

## `InitTextureEx` / `InitTexture`

**Contract** — loads nine pieces by suffixing the base name — interior, four edges, four
corners — into one shader and nine rectangles, then checks nine dimensional-agreement rules.
Reports whether every piece was found. In fatal mode each missing piece and each broken rule
is an immediate failure; otherwise the rules are reported as diagnostics and the frame is
simply marked absent.

**Notes** — the missing-piece check runs in *both* modes even though the source is shaped as
if it were fatal-only, so a non-fatal load still reports correctly. That was a deliberate
change, recorded in the source, to keep debug builds playable against incomplete art.

Two of the eleven possible agreement rules — the interior against the top edge's width and
against the left edge's height — are commented out. The interior tiles in both directions, so
its size is genuinely free; the rules were over-strict and their removal is a decision, not an
oversight.

## `DrawElements`

**Contract** — emits the whole frame as one primitive: the tile count is computed first so the
vertex budget is exact, then the four corners, then the four edges, then the interior.

```text
FUNCTION draw_elements()
  bind the single shader
  rect <- absolute rect converted to screen pixels
  spare <- (rect width  - left corner width  - right corner width,
            rect height - top corner height  - bottom corner height)
  REQUIRE spare is non-negative in both axes

  quads <- 4                                            # the corners
  IF spare.x > 0 THEN quads <- quads + 2 * ceil(spare.x / top edge width)
  IF spare.y > 0 THEN quads <- quads + 2 * ceil(spare.y / left edge height)
  IF both > 0    THEN quads <- quads + ceil(spare.x / interior width)
                                     * ceil(spare.y / interior height)
  begin a primitive of quads * 6 vertices

  emit the four corners, each at its own corner of the rect
  IF spare.x > 0 THEN tile the top edge and the bottom edge between the corners
  IF spare.y > 0 THEN tile the left edge and the right edge between the corners
  IF both > 0    THEN tile the interior across the inner rectangle
  flush
```

**Notes** — computing the exact vertex count before emitting anything is the reason the whole
frame is one primitive, and it is why the tile counts are computed twice (once to count, once
to emit). A rebuild with a growable vertex buffer can count once.

The counting has a latent bug worth naming: each of the three contributions is accumulated
into one reused variable, and only the last one computed survives to be added when an earlier
branch did not run. In the shipped layouts every panel has spare space in both axes, so all
three branches run and the count is right; a panel that stretched in only one axis would
under-count and, on a device that validates the budget, fail. Do not reproduce the reuse.

## `get_points` — the clipping rule

**Contract** — produces one quad's screen corners and texture corners for a piece placed at a
rectangle's top-left, *trimming both together* when the piece does not fit.

```text
FUNCTION get_points(area, piece) -> screen_lt, screen_rb, tex_lt, tex_rb
  tex_lt, tex_rb <- the piece's source rect
  screen_lt      <- area top-left
  screen_rb      <- screen_lt + the piece's source extent

  overflow_x <- area width  - piece width
  overflow_y <- area height - piece height
  IF overflow_x < 0 THEN trim both screen_rb.x and tex_rb.x by that amount
  IF overflow_y < 0 THEN trim both screen_rb.y and tex_rb.y by that amount
```

**Notes** — this is the whole reason the tiling looks right: the last tile in a run is *cut*,
in source and destination equally, rather than scaled. It is the same decision the frame line
makes for its partial tile, expressed differently, and a rebuild should share one helper.

## `draw_tile_line` / `draw_tile_rect`

**Contract** — repeat one piece along an axis until the area is filled, advancing the cursor to
each emitted quad's far edge; and repeat that line across the other axis for the interior. Both
terminate on a small epsilon so a tile boundary landing exactly on the edge does not emit a
zero-width quad.

**Notes** — the advance uses the *emitted* quad's far edge rather than the piece's nominal
size, which is what makes the trimmed last tile terminate the loop instead of looping forever.
The interior's outer loop clamps its own cursor to the area's far edge for the same reason.

## `Draw`

**Contract** — the frame first when a texture is loaded, then the child walk, so children draw
inside the frame.

## `GetTitleText`

**Contract** — an optional caption, created and attached on first request.

## `SetTextureRect` / `GetTextureRect` / `SetStretchTexture` / `GetStretchTexture` / `SetTextureColor` / `GetTextureColor`

**Contract** — the texture-owner interface, mostly inert. As with the frame line, a nine-piece
frame has no single texture rectangle: the setter does nothing, the getter answers the
interior's. Stretching is meaningless — the pieces always tile — so the setter does nothing
and the query answers no. Only the colour is real.

**Notes** — three of these six are stubs, which is the strongest evidence in the chapter that
the texture-owner interface was drawn around the flat picture widget and then imposed on
widgets that do not fit it. A rebuild should let a widget declare *which* texture protocol it
speaks instead.
