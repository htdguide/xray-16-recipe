# src/xrUICore/Windows/UIFrameLineWnd.cpp

> Draws a three-piece stretchable line — a fixed cap at each end and a middle tiled to fill — from one texture, in either orientation.

**Needs** — [`UIFrameLineWnd.h`](UIFrameLineWnd.h.md) · [`UIWindow.h`](UIWindow.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`ui_base.h`](../ui_base.h.md) · [`Include/xrRender/UIRender.h`](../../Include/xrRender/UIRender.h.md) · [`Include/xrRender/UIShader.h`](../../Include/xrRender/UIShader.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIFrameLineWnd.h`](UIFrameLineWnd.h.md)
**Tier floor** — T1: it builds a triangle list with explicit positions, texture coordinates and colours and submits it to the device, batching by shader.

## Purpose

A great deal of this UI is lines that must stretch: a scroll bar's track and thumb, a
progress bar's fill, a separator, the frame of a spin box. Stretching a single texture
distorts the decorated ends, so the engine's convention is three textures — a start cap, a
tiling middle, an end cap — named by suffixing one base name. This file is that drawer.

It is also the reason the toolkit needs so few widget types: any widget that wants a
stretchable appearance derives from this one and gets it, in either orientation, for free.

## State

```text
RECORD FrameLineWnd EXTENDS Window, TextureOwner
  segment_rect  : Rect   x 3      # first cap, middle, second cap: source rects in the atlas
  segment_shader: Shader x 3      # usually all three are the same
  texture_colour: colour          # opaque white
  texture_shown : bool            # false until a successful load
  stretch       : bool            # true: the line fills the widget's short dimension
  horizontal    : bool            # true
  title         : optional<Static>  # a label, created only on demand
```

**Invariants**

- `texture_shown` is true only when **all three** pieces loaded. A partial set draws nothing
  rather than something broken — which is what lets callers probe a texture name and fall
  back to another asset generation.
- The three source rectangles are in atlas pixels; they are divided by the atlas resolution at
  emit time, never stored normalized.
- The caps are *fixed* size and the middle is the only piece that repeats. A rebuild that
  stretches the caps will distort every framed control in the game.

## `InitTextureEx` / `InitTexture`

**Contract** — loads the three pieces by suffixing the base name — the middle with one
suffix, the two caps with another two — and reports whether all three were found. Fatal only
when the caller asked for fatal; otherwise it returns false and leaves the line invisible, so
a caller can try a different name. Warns, in non-shipping builds, when the pieces disagree on
the dimension that must match: height for a horizontal line, width for a vertical one.

**Notes** — the cross-dimension agreement is a real constraint, not a style check. The drawer
takes the line's thickness from the *first cap's* source rectangle when not stretching, so a
middle of a different height silently draws at the cap's height and scales.

## `Draw`

**Contract** — draws the three-piece line when a texture is loaded, then the child walk. So
the line is a background and anything attached to it draws on top.

## `DrawElements` — the layout and emission

**Contract** — converts the widget's rectangle to screen pixels, works out how many middle
tiles fit, and emits one quad per piece, batching all of them into a single primitive when the
three pieces share a shader.

```text
FUNCTION draw_elements()
  rect <- absolute rect converted to screen pixels
  first_len, second_len, back_len <- the three source rects' extents along the line's axis,
                                      converted to screen pixels

  available <- rect length - first_len - second_len
  IF available > 0 AND back_len > 0
    tiles_exact <- available / back_len
    whole_tiles <- floor(tiles_exact)
    remainder   <- tiles_exact - whole_tiles
  ELSE IF available < 0
    shrink the rect by one middle tile      # the line is shorter than its own caps
```

```text
  one_shader <- all three segments share a shader
  begin a primitive sized for (one_shader ? every tile : just the first cap)
    emit the first cap at the running cursor
  IF NOT one_shader THEN flush and begin a primitive sized for the middle
    emit whole_tiles copies of the middle
    IF remainder > 0
      emit one more with its source rect trimmed to the remaining fraction
  IF NOT one_shader THEN flush and begin a primitive sized for one quad
    emit the second cap
  flush
```

Each quad's cross-axis extent is the widget's own when stretching, and the source rectangle's
own when not — which is the difference between "fill this box" and "draw at the art's natural
thickness along this box's edge".

**Notes** — several decisions here are worth stating plainly.

The *partial* tile at the end is drawn with its source rectangle trimmed to the same fraction,
so the middle is cut rather than squashed. This is the difference between a clean fill and a
visibly compressed last tile, and it is the one thing most reimplementations get wrong.

The batching is a real optimization with a visible constraint: when all three pieces come from
one atlas — which they do whenever they were loaded from the same texture — the whole line is
one primitive, so a scroll bar's track costs one draw. When they differ, the line costs three.
A rebuild should keep the single-batch path; the three-batch path exists only for the
adapted assets that copy a piece's shader in from elsewhere.

The negative-space case — a widget shorter than its two caps — shrinks the rectangle by a
middle tile's length and then draws the caps overlapping. It is a degenerate case with no
good answer; reproduce it or clamp the widget's minimum size, but know that the shipped
layouts do produce it.

The horizontal path multiplies the second cap's length by the screen's aspect correction and
the first cap's length not at all. That asymmetry has no discoverable reason and is visible as
a slightly wider right cap on a wide display. Recorded as unrecovered.

Every emitted corner is snapped to a pixel and then shifted by half a pixel up and left. The
half-pixel is the classic texel-centre correction for this era's rasterization rules: without
it, a one-pixel border samples between texels and blurs. A rebuild on a modern API may not
need the shift, but must then check the frame art still reads crisply.

## `SetTextureRect` (by segment) / `SetShader` / `SetTextureVisible` / `SetTextureColor` / `SetStretchTexture` / `SetHorizontal`

**Contract** — direct writes to the three-segment description. Setting the shader sets all
three at once — the common case — while setting a rectangle names a segment. Writing a
segment out of range is refused rather than trusted.

**Notes** — the texture-owner interface's single-rectangle setter is deliberately **inert**
here: a three-piece line has no one texture rectangle, and silently accepting a write would
half-configure it. Its matching reader answers the first cap's rectangle. That pair is the
seam where a generic texture-owner interface meets a widget it does not fit, and the honest
resolution in a rebuild is a separate interface for segmented art.

## `GetTitleText`

**Contract** — the optional label, created and attached on first request. Lines used as frames
carry a caption; lines used as scroll-bar parts do not, and pay nothing for the possibility.

## `FillDebugTree` / `FillDebugInfo`

**Contract** — exposes the colour, the three flags and the three source rectangles — as
origin-plus-size, clamped to the atlas — to the debug overlay, which is how the shipped art's
rectangles were tuned. Non-shipping builds only.
