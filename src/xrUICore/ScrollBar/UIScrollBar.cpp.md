# src/xrUICore/ScrollBar/UIScrollBar.cpp

> The scroll bar — two stepper buttons, a draggable thumb and a track — and the two-way mapping between an integer scroll position and the thumb's pixel position.

**Needs** — [`UIScrollBar.h`](UIScrollBar.h.md) · [`UIScrollBox.h`](UIScrollBox.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Seam: Debug overlay UI](../../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`UIScrollBar.h`](UIScrollBar.h.md)
**Tier floor** — T3: integer/pixel mapping and a small input protocol; the only sharp edge is that the mapping must round the same way in both directions.

## Purpose

Scrolling in this toolkit is expressed in two different units and the scroll bar is the
translator. The *view* thinks in pixels of the toolkit's virtual screen: the content pad is
at some negative offset. The *bar* thinks in an integer position between a minimum and a
maximum, with a page size carved out of the range for the visible window. Every other widget
that scrolls — the scroll view, the list, the combo drop-down — talks to the bar in the
integer units and never computes a thumb position itself.

The file also owns the composition: a scroll bar is four children, not a drawn control.

## State

```text
RECORD ScrollBar EXTENDS Window
  dec_button   : Button          # up / left stepper
  inc_button   : Button          # down / right stepper
  thumb        : ScrollBox       # the draggable box
  track        : FrameLineWnd    # the background the thumb rides in
  horizontal   : bool
  min_pos      : int             # 1
  max_pos      : int             # 1
  page_size    : int             # 1; how much of the range the view shows at once
  step_size    : int             # 1; one stepper click
  scroll_pos   : int
  work_area    : int             # track length in pixels: bar length minus both steppers
  hold_delay   : real            # ms between repeats while a button is held
  mouse_state  : ENUM { idle, decrementing, incrementing }
  enabled      : bool            # a disabled bar can never be shown
```

**Invariants**

- `scroll_pos` lies in `min_pos .. max_pos - page_size + 1`, clamped on every write. The
  upper bound is the *top* of the last page, not the last item.
- `scroll_span = max(1, max_pos - min_pos - page_size + 1)` — the number of distinct
  positions. It is floored at one so the mapping never divides by zero on a range that fits
  entirely in one page.
- `max_pos` is bumped by one whenever it equals `min_pos`, for the same reason.
- `mouse_state` exists so that a press that started on one stepper cannot, by sliding, also
  drive the other: the opposite direction is refused while a direction is latched.
- The two stepper buttons are explicitly *unregistered* from the focus system. Focus
  navigation with a gamepad must land on the thing being scrolled, never on the bar's arrows;
  the owner of the bar scrolls it.

## `InitScrollBar`

**Contract** — builds the bar from a named profile in the shipped scroll-bar layout: the
thickness, the repeat delay, the two arrow buttons, the thumb art and the track art. Returns
whether every piece was found in its modern form. Not fatal on a missing piece — the older
games describe the thumb and track as plain statics rather than three-part stretchable lines,
and this routine adapts.

```text
FUNCTION init_scroll_bar(position, length, horizontal, profile) -> bool
  doc <- load ui layout "scroll_bar"
  thickness <- doc[profile].height, or .height_v when vertical, defaulting to 16
  hold_delay <- doc[profile].hold_delay, default 50 ms

  size self: horizontal ? (length, thickness) : (thickness, length)
  configure dec_button from doc[profile][leading arrow]   ; place at the leading corner
  configure inc_button from doc[profile][trailing arrow]  ; place at the trailing corner

  # thumb and track: try the three-part stretchable form first
  IF NOT configure_frame_line(thumb, doc[profile][box element])
     probe <- a throwaway static configured from the same element
     IF probe loaded
       copy probe's shader and texture rectangle into the thumb's
       centre segment, and blank the two end segments
  same probe-and-adapt for the track, whose texture name is read directly

  IF the track adapted from a static
    adopt the probe's thickness as the bar's own thickness

  work_area <- max(0, bar length - 2 * thickness)
  lay out the thumb
  RETURN false IF either piece had to adapt
```

**Notes** — the probe-and-adapt path is the entire compatibility story with the two earlier
games, and it is worth stating as a decision rather than a workaround: *the thumb and the
track are always three-segment stretchable lines internally*; a single-image asset is loaded
into the centre segment with the ends blanked, which renders identically because a blank end
segment has zero length. A rebuild therefore needs only the three-segment drawer, plus this
one adapter at load.

The return value reports "everything was in the modern form", which the scroll view uses to
decide whether its fixed-geometry variant is usable at all — so a `false` here is a routing
signal, not an error.

## `UpdateScrollBar`

**Contract** — recomputes the thumb's size and position from the range, the page size and the
current position, repositions the trailing stepper and the track, and makes the thumb
invisible when the content fits. Called after every change to any of its inputs.

```text
FUNCTION update_scroll_bar()
  IF shown
    IF max_pos == min_pos THEN max_pos <- max_pos + 1
    thumb_len <- work_area * max(page_size, 1) / (max_pos - min_pos)
    IF scrollable
      clamp thumb_len between min(bar thickness, track length) and track length
      set thumb's long dimension to thumb_len and its short dimension to the bar's
      thumb offset <- position_to_pixels(thumb_len, bar thickness)
      pin the trailing stepper to the trailing corner
      stretch the track to fill between the two steppers

  thumb.texture_alpha <- scrollable ? opaque : transparent
  clamp_thumb_into_track()
```

**Notes** — "not scrollable" is expressed by making the thumb *transparent*, not by hiding it.
The thumb keeps its place in the tree and keeps consuming mouse events over its rectangle; a
rebuild that hides it instead will change which widget a click lands on. Whether that is
intentional is not recoverable; it is observable.

The thumb's minimum length is the bar's own thickness — a square thumb — so a very long
document still leaves something grabbable.

## `PosViewFromScroll` and `SetPosScrollFromView`

**Contract** — the two halves of the mapping. One converts the integer position into the
thumb's pixel offset along the track; the other converts a dragged thumb offset back into an
integer position. Both are total: a degenerate track maps to zero.

```text
FUNCTION position_to_pixels(thumb_len, lead_offset) -> int
  travel <- work_area - thumb_len            # pixels the thumb can move
  span   <- scroll_span
  IF span == 0 RETURN 0
  RETURN (scroll_pos * travel + span * lead_offset - min_pos * travel) / span

FUNCTION pixels_to_position(thumb_offset, thumb_len, lead_offset)
  travel <- work_area - thumb_len
  IF travel == 0 THEN set_position_clamped(0) ; RETURN
  set_position_clamped(floor((thumb_offset - lead_offset) / travel * scroll_span + min_pos))
```

**Notes** — `lead_offset` is the leading stepper's size: the thumb's coordinates are relative
to the bar, so the first stepper's length is a constant bias that must be added on the way out
and removed on the way in. The forward map does the whole computation in integers, which means
the thumb snaps to positions rather than gliding — visible on a long list as a thumb that
moves in small jumps, and faithful.

The two directions are not exact inverses (one floors a division, the other rounds by integer
arithmetic on a different expression), so a drag followed by a redraw can move the thumb by a
pixel. This is why the drag path calls the clamp routine immediately afterwards.

## `ClampByViewRect`

**Contract** — pushes the thumb back inside the track after any move: it may not start before
the leading stepper ends, and it may not end after the trailing stepper begins.

**Notes** — this is the only guard on the thumb's position; the drag path adds a raw cursor
delta and relies on this to keep it legal.

## `OnMouseDownEx`

**Contract** — the hit test for a press anywhere on the bar. Resolves the cursor against five
regions and acts: the two steppers step by one; the two gaps between a stepper and the thumb
page by four steps; the thumb itself is handled by the thumb widget. Returns whether the press
was claimed. Latches a direction so a held press cannot reverse.

```text
FUNCTION on_mouse_down_ex() -> bool
  cursor <- absolute cursor position
  IF cursor in dec_button AND mouse_state != incrementing
    try_scroll_dec() ; mouse_state <- decrementing ; RETURN true
  IF cursor in inc_button AND mouse_state != decrementing
    try_scroll_inc() ; mouse_state <- incrementing ; RETURN true

  # the two page regions: between the leading stepper and the thumb,
  # and between the thumb and the trailing stepper
  IF cursor in leading gap  AND mouse_state != incrementing
    try_scroll_dec(page = true) ; mouse_state <- decrementing ; RETURN true
  IF cursor in trailing gap AND mouse_state != decrementing
    try_scroll_inc(page = true) ; mouse_state <- incrementing ; RETURN true
  RETURN false
```

## `OnKeyboardAction`

**Contract** — on a *held* left mouse button, re-runs the press hit test once every
`hold_delay` milliseconds. This is the auto-repeat.

**Notes** — the repeat lives on the keyboard path because the toolkit delivers "this key is
still down" as a keyboard hold event and treats mouse buttons as keys with reserved codes.
The last-repeat timestamp is a single module-wide value, not per-bar, so two bars held at once
would share one repeat clock. In practice only one can be held; a rebuild should still make
it per-instance.

## `ScrollInc` / `ScrollDec` / `TryScrollInc` / `TryScrollDec`

**Contract** — the step primitives. A step moves by `step_size`, or by four times that when
the step came from a page region or the wheel. The `Try` forms step and, if anything moved,
notify the message target with the horizontal or vertical scroll message — which is how the
owning view learns to move its content.

**Notes** — decrementing has a special case: when the position is at or below one step from
the bottom, it goes to *zero* rather than to `min_pos`. Since `min_pos` defaults to one, this
lets a bar reach a position its own clamp would otherwise forbid, and the next clamp pulls it
back. The factor of four for page and wheel scrolling is a feel constant with no derivation.

## `OnMouseAction`

**Contract** — the wheel scrolls by a page in the matching direction and is always consumed;
a left release clears the latched direction; everything else goes to the children, which is
how the thumb receives its drag.

## `SendMessage`

**Contract** — the bar's own children report to it. A click or press on either stepper steps.
A move reported by the thumb is the drag: clamp the thumb, convert its pixel offset back into
a position, and notify the owner with the scroll message.

**Notes** — reacting to *both* the click and the press message from a stepper means a single
click steps once on press and again on release. That doubling is observable in the shipped
game and a rebuild that "fixes" it changes the feel of every list.

## `SetRange` / `SetPageSize` / `SetStepSize` / `SetScrollPos` / `SetWidth` / `SetHeight` / `Refresh`

**Contract** — the setters; each recomputes the layout. Resizing recomputes the track length
from the new size minus both steppers and floors it at zero. A zero or negative size is
silently promoted to one, because the mapping divides by the track length.

`Refresh` re-runs the drag path against the thumb's current position — the way a caller forces
the owner to be re-notified without the user touching anything.

## `Show` / `Enable` / `IsRelevant` / `Reset`

**Contract** — a bar marked disabled refuses to be shown or enabled at all and hides itself
when disabled; this is a hard off-switch distinct from the window's ordinary enabled flag.
`IsRelevant` answers whether there is anywhere to scroll in either direction, and is what the
thumb's visibility and the layout both key off. `Reset` resets every child and then itself.

## `FillDebugInfo`

**Contract** — exposes the range, page, step, position, track length and orientation to the
debug overlay, re-laying out and re-notifying when the position is dragged there. Present only
in non-shipping builds.
