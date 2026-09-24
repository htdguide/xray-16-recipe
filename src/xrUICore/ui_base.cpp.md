# src/xrUICore/ui_base.cpp

> The module's single runtime object — it owns the canvas-to-screen scale, the scissor stack, the clipping frustums, the font manager, the cursor, the focus system and the debug overlay, and it is the only place in the chapter that knows the real display resolution.

**Needs** — [`ui_base.h`](ui_base.h.md) · [`ui_defs.h`](ui_defs.h.md) · [`ui_focus.h`](ui_focus.h.md) · [`ui_debug.h`](ui_debug.h.md) · [`FontManager/FontManager.h`](FontManager/FontManager.h.md) · [`Cursor/UICursor.h`](Cursor/UICursor.h.md) · [`XML/UIXmlInitBase.h`](XML/UIXmlInitBase.h.md) · [`XML/UITextureMaster.h`](XML/UITextureMaster.h.md) · [`xrCore/XML/XMLDocument.hpp`](../xrCore/XML/XMLDocument.hpp.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ui_base.h`](ui_base.h.md)
**Tier floor** — T1: it converts canvas units to device pixels and hands integer scissor
rectangles to the graphics device, and it is on the per-quad path.

## Purpose

Everything else in the chapter works in a fixed 1024×768 canvas and knows nothing about the
display. This file is where that fiction is maintained: it holds the scale from canvas to
device pixels, rebuilds it whenever the device resets, and offers the conversions every draw
path calls. It also owns the four module-wide services (fonts, cursor, navigation focus,
debug overlay) and the scissor stack, because all of them need the scale and none of them
should own a second copy of it.

It is reached through a global — the instance registers itself in the engine's service
locator and every `UI()` in the codebase is a reach for it. That is the cycle-breaker
described in §7 of the system requirements; a rebuild injects it instead.

## State

```text
RECORD UICore
  scale            : vec2          # device pixels per canvas unit, normal path
  pp_scale         : vec2          # ditto, while drawing into the post-process pass
  current_scale    : reference to one of the two above
  point_type       : PointType     # TL (screen-space, scaled) or LIT (world-space, unscaled)
  postprocess      : bool

  frustum          : Frustum2D     # clip region for the normal path: the whole back buffer
  frustum_pp       : Frustum2D     # ditto for the post-process path
  frustum_lit      : Frustum2D     # clip region for world-space UI (a computer screen in the world)

  scissors         : stack<Rect>   # in canvas units; the top is the effective clip

  fonts            : FontManager
  cursor           : Cursor
  focus            : FocusSystem
  debugger         : Debugger
```

**Invariants**

- `current_scale` always points at one of the two scale vectors; the post-process bracket is
  the only thing that moves it, and it must be balanced.
- Every rectangle on the scissor stack is inside the canvas. Pushing a rectangle that is not
  is a bug loud enough to log both the request and the result before it fails.
- When `point_type` is LIT, **no scaling and no scissor happen at all**: the coordinates are
  already in the space the caller wants and the toolkit must not touch them.
- On a dedicated server there is no cursor and no font manager; both are absent, not empty.

## `UI()` and `GetUICursor()`

**Contract** — the two reach-throughs to the global instance. `UI()` returns the module
object; `GetUICursor()` is a shorthand for its cursor because half the widget code wants
only that. Neither allocates and neither can fail once startup has run.

## `UICore` construction and destruction

**Contract** — builds the cursor and font manager unless this process is a dedicated server,
then performs a device reset and a UI reset to establish the scale and load the shared
texture registry. Destruction releases fonts and cursor and frees both the colour
definitions and the texture registry.

**Notes** — the two resets are called from the constructor, which means the display must
already be sized when the module comes up. That is the actual ordering requirement between
this chapter and the device.

## `OnDeviceReset`

**Contract** — recomputes the canvas-to-pixel scale from the current back-buffer size and
rebuilds the screen clipping frustum to cover the whole back buffer. Called by the engine
whenever the resolution or the device changes.

```text
FUNCTION on_device_reset(ui)
  ui.scale = (device_width / CANVAS_WIDTH, device_height / CANVAS_HEIGHT)
  ui.frustum = frustum_from_rect(0, 0, device_width, device_height)
```

**Notes** — the scale is non-uniform. A 16:9 display stretches the canvas horizontally, and
the engine *accepts* that distortion for layout while correcting for it in two places: the
aspect factor applied to rotations (`get_current_kx`), and the separate `_16` variants of the
layout files (`get_xml_name`). A rebuild that letterboxes instead would look different from
the original and fail the "recognizably the same image" criterion.

## `OnUIReset`

**Contract** — throws away and reloads everything that came from data: the named colour
definitions and the whole texture registry. Called when the UI style changes or when a
reload is requested. Idempotent.

**Notes** — the order is: free colours, free textures, re-read textures, re-read colours.
The re-read of textures must happen before colours because the colour file is looked up
through the same path-with-style-override machinery, and freeing first guarantees the new
style's definitions override rather than merge with the old ones.

## `ReadTextureInfo`

**Contract** — populates the shared texture registry by scanning the configuration tree for
texture-description documents. Reads every `*.xml` under the default style's
`textures_descr` directory, then — if a non-default style is active — every one under that
style's directory too, so a style overrides individual entries without having to restate the
whole set. Finally, an explicit list of extra description files may be named in the engine
configuration and is loaded last.

```text
FUNCTION read_texture_info(ui)
  parse every "<ui_path_default>/textures_descr/*.xml"
  IF the active style path differs from the default
    parse every "<ui_path>/textures_descr/*.xml"       # later wins
  IF configuration has a "texture_desc" section
    FOR EACH name IN its "files" list
      parse "<name>.xml"
```

**Notes** — "later wins" is how a style is built: a style ships only the entries it changes.
The configuration-driven extra list is how a total-conversion mod adds pages without editing
the shipped directories.

## `ClientToScreenScaled` and friends

**Contract** — the four conversions every draw path calls: point, width, height, and
pixel-alignment. All of them are identity when the current point type is LIT. `AlignPixel`
floors a coordinate, which snaps a widget's edge to a pixel boundary so a 1-pixel border
does not blur across two pixels.

```text
FUNCTION to_screen(ui, p)        -> p * ui.current_scale        # unless LIT
FUNCTION to_canvas_width(ui, w)  -> w / ui.current_scale.x      # the inverse
FUNCTION align_pixel(ui, v)      -> floor(v)                    # unless LIT
```

**Notes** — the width/height conversions divide where the point conversion multiplies, and
both are named "ClientToScreenScaled…". They go in opposite directions. The naming is wrong
in the original; the *behaviour* is what matters — callers use the divide form to turn a
measured pixel extent (a font's measurement of a string, say) back into canvas units.

## `PushScissor` / `PopScissor`

**Contract** — maintains a stack of clip rectangles in canvas units and keeps the graphics
device's scissor in step with the top of the stack. A push normally intersects with the
current top, so nesting narrows; passing the "overlapped" flag intersects with the whole
canvas instead, which is how a popup escapes its parent's clip. An empty intersection becomes
a degenerate rectangle rather than an error, so the contents simply vanish. Both operations
are no-ops in LIT mode.

```text
FUNCTION push_scissor(ui, r, overlapped)
  IF ui.point_type IS LIT THEN RETURN
  bound = overlapped OR stack is empty ? whole canvas : top of stack
  result = intersect(bound, r) ELSE empty rectangle
  assert result lies inside the canvas
  push result
  set device scissor to result scaled to pixels, floored at the left/top
      and rounded outward at the right/bottom

FUNCTION pop_scissor(ui)
  IF ui.point_type IS LIT THEN RETURN
  pop; set device scissor to the new top, or disable it if the stack is now empty
```

**Invariants** — pushes and pops must balance within a frame; an unbalanced push leaves the
device scissor clamping the next frame's drawing.

**Notes** — the right and bottom edges are rounded *outward* (floor of value + ½) while left
and top are floored. Rounding both the same way would drop the last column of pixels of a
widget whose right edge lands mid-pixel.

## `pp_start` / `pp_stop`

**Contract** — brackets drawing that goes into the post-process pass rather than straight to
the back buffer. Inside the bracket the alternate scale and frustum are current and the font
layer is told its own scale. Must be balanced.

**Notes** — in the current code the post-process scale is computed identically to the normal
one and the font scale is set to a literal 1:1 (expressed as `width/width`), so the bracket
changes nothing measurable. It is a seam left open for a post-process pass at a different
resolution than the back buffer. A rebuild may collapse it; if it does, it must keep the
frustum switch, because the post-process target may legitimately differ in size.

## `is_widescreen` / `get_current_kx`

**Contract** — `is_widescreen` reports whether the display is meaningfully wider than the
canvas's 4:3 aspect, with a small tolerance so that a display that is 4:3 to within a
rounding error is not misclassified. `get_current_kx` returns the factor by which the canvas
is horizontally stretched — the correction a rotation must apply so a circle stays a circle.

```text
FUNCTION is_widescreen() -> bool
  RETURN device_width / device_height > (CANVAS_WIDTH / CANVAS_HEIGHT) + 0.01

FUNCTION get_current_kx() -> real
  RETURN (device_height / device_width) / (CANVAS_HEIGHT / CANVAS_WIDTH)
```

**Notes** — the 0.01 tolerance is a guard against floating-point noise on nominally 4:3
modes; nothing in the data depends on its exact value. `get_current_kx` is below 1 on a wide
display, which shrinks rotated art horizontally to undo the canvas stretch.

## `get_xml_name`

**Contract** — resolves a layout document name against the display aspect: on a widescreen
display, prefer a `<name>_16.xml` variant if it exists in the given path, otherwise fall back
to `<name>.xml`. Returns the name to open. Does not open anything.

```text
FUNCTION get_xml_name(path, name) -> text
  IF NOT is_widescreen() THEN RETURN name with ".xml" appended if it has no extension
  candidate = name with its extension replaced by "_16.xml"
  IF candidate exists under `path` THEN RETURN candidate
  RETURN name with ".xml" appended if it has no extension
```

**Notes** — this is the engine's entire answer to widescreen layout: ship a second hand-
authored copy of each screen. It is why the layout data is **frozen in form** — the `_16`
suffix convention is part of the data contract, not an implementation detail. A rebuild with
real responsive layout still has to honour the suffix to load the shipped screens correctly.

## `RenderFont`

**Contract** — flushes the font layer's accumulated text for the frame. Text is batched
across the whole UI and emitted once at the end, rather than per widget, because each font
page is a separate material and interleaving them with widget quads would multiply the
material switches. That batching is the reason text draws *on top of* everything, regardless
of tree order — a constraint a rebuild inherits along with the batching.
