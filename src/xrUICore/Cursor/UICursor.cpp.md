# src/xrUICore/Cursor/UICursor.cpp

> Keeps one pointer position in virtual UI coordinates, fed either by relative deltas or by the OS pointer depending on capture mode, and draws the cursor and the deferred hints after every other UI element.

**Needs** — [`UICursor.h`](UICursor.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Buttons/UIBtnHint.h`](../Buttons/UIBtnHint.h.md) · [`ui_base.h`](../ui_base.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UICursor.h`](UICursor.h.md)
**Tier floor** — T1: pixel-to-virtual conversion in both directions, a display-bounds query, and a write to the OS pointer.

## Purpose

Two decisions live here and both are easy to get wrong in a rebuild.

The first is that the cursor is *the* hit-test position for the whole toolkit: widgets do not
receive pointer coordinates from the platform, they compare their rectangle against this
position. So this position must be in the same space as widget rectangles, which is the
virtual 1024×768 space, and the conversion factor is the *window client* size, not the
back-buffer size.

The second is that there are two input regimes and they cannot be merged. When the pointer is
captured — exclusive mode, or a window larger than the desktop — motion arrives as relative
deltas and the position is integrated here. When it is not, the OS pointer is authoritative
and its absolute position is read and converted. Integrating deltas in the second case would
drift away from the visible system pointer.

## State

```text
RECORD Cursor
  position      : point (virtual UI units, clamped to 0..1024 × 0..768)
  previous      : point
  correction    : point   # virtual units per client pixel, per axis
  visible       : bool
  bound_to_system_cursor : bool
  image         : Static  # the drawn cursor quad
```

**Invariants**

- `position` is clamped to the virtual screen after every update, so a widget at the extreme
  edge is always reachable and the position is never outside the hit-test space.
- `correction` is recomputed whenever the device resets; it is `virtual_extent /
  client_pixel_extent` per axis, so multiplying a pixel position by it yields virtual units
  and dividing does the reverse.
- The cursor is drawn exactly once per frame; a second draw in the same frame is a bug the
  debug build asserts on.

## Construction and reset

**Contract** — builds the cursor image from a fixed texture and shader pair, gives it a
40×40-unit source rectangle, and sets its drawn width to 40 scaled by the aspect correction
factor so that the cursor is not stretched on a wide screen, then registers itself in the
device's render sequence at a priority that places it after the rest of the UI. A UI reset
(a style or language change) destroys and rebuilds the image; a device reset only recomputes
the conversion factors.

```text
FUNCTION on_device_reset()
  correction <- (VIRTUAL_WIDTH  / client_width,
                 VIRTUAL_HEIGHT / client_height)

  # when the window is no larger than the desktop, the OS pointer can reach
  # every part of it, so the OS pointer can be the authority
  desktop <- display_bounds(primary)
  bound_to_system_cursor <- desktop.height >= backbuffer_height
                        AND desktop.width  >= backbuffer_width
```

**Notes** — the 40×40 size is the cursor art's own size and is hard-coded rather than read
from the texture description, so replacing the cursor art with a different size changes
nothing until this constant changes too. The aspect correction is applied to width only,
which keeps the cursor square on screen when the virtual space is stretched horizontally.

## `update_cursor_position`

**Contract** — called once per frame with the platform's motion report. In captured mode the
argument is a relative delta and is scaled into virtual units and accumulated; otherwise the
OS pointer is queried and its absolute pixel position converted. Either way the result is
clamped to the virtual screen, and the previous position is kept so that a delta can be
reported.

```text
FUNCTION update_cursor_position(motion)
  previous <- position
  IF input.exclusive_mode OR NOT bound_to_system_cursor
    position <- position + motion * correction     # relative
  ELSE
    position <- os_pointer_pixels() * correction   # absolute
  position <- clamp(position, screen)
```

**Notes** — the sensitivity multiplier in the relative branch is 1 and is written out
explicitly; there is no cursor speed setting.

## `set_ui_cursor_position`

**Contract** — teleports the cursor to a virtual position *and* moves the OS pointer to the
corresponding pixel, so the two never separate. The platform's report of the move is ignored.

## `warp_to_window`

**Contract** — places the cursor on a widget. With no widget, centres it on the virtual
screen. Centred on a widget when asked; otherwise at two-thirds across and two-thirds down
the widget's box — inside it, past its centre, and clear of the widget's own top-left corner
where an adjacent widget's rectangle might also be.

**Notes** — this is the bridge that makes keyboard and gamepad navigation work at all: the
toolkit has no focus-driven dispatch for the pointer, so moving focus means moving the
pointer. Every directional-navigation path ends in a call here.

## `on_render`

**Contract** — runs at the end of the frame's UI rendering. Draws the two deferred hint boxes
first, then — only if visible — updates and draws the cursor image at the current position.
An invisible cursor still lets the hints draw, which is why they are rendered before the
visibility test.

**Invariants** — this is the last UI draw of the frame, which is what makes both the cursor
and the hints appear above everything regardless of the window tree.
