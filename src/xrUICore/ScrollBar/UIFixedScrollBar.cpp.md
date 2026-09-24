# src/xrUICore/ScrollBar/UIFixedScrollBar.cpp

> A scroll bar whose dimensions are dictated by its art rather than its owner, and which drags its thumb itself instead of letting the thumb drag itself.

**Needs** — [`UIFixedScrollBar.h`](UIFixedScrollBar.h.md) · [`UIScrollBar.h`](UIScrollBar.h.md) · [`UIScrollBox.h`](UIScrollBox.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`XML/UIXmlInitBase.h`](../XML/UIXmlInitBase.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [Data: UI layout and text](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UIFixedScrollBar.h`](UIFixedScrollBar.h.md)
**Tier floor** — T3: the same mapping as the ordinary bar, plus an inset.

## Purpose

The third game's personal-data-assistant screens use scroll bars whose art is drawn to a
specific size and inset from the track edges — the thumb does not touch the arrows, and the
whole bar cannot be stretched to whatever height its container happens to be. This file is
that variant.

It exists as a separate type rather than a flag because three things change together: the
size comes from the profile instead of the caller, the thumb is a three-state button rather
than a stretchable line, and every position computation gains a per-axis inset. Only the
range model and the step primitives are shared with
[`UIScrollBar`](UIScrollBar.cpp.md).

## State

```text
RECORD FixedScrollBar EXTENDS ScrollBar
  thumb          : Button       # shadows the base bar's thumb, which stays unused
  thumb_inset    : (int, int)   # gap between the thumb and each stepper, per axis
```

**Invariants** — the base class's own thumb still exists and is still attached to the tree; it
is simply never positioned or drawn by this variant. A rebuild should have one thumb slot,
typed as whatever both variants can drive.

## `InitScrollBar`

**Contract** — builds the bar entirely from the named profile: four dimensions (width and
height, and a second pair for the vertical orientation), the repeat delay and the two inset
values, then the two steppers, the thumb button and the three-segment track. Fails — returning
false without asserting — if the track art is missing, which tells the scroll view to fall
back to the ordinary bar. Unregisters the thumb from focus navigation for the same reason the
base class unregisters the steppers.

```text
FUNCTION init_scroll_bar(position, horizontal, profile) -> bool
  doc <- load ui layout "scroll_bar"
  read width, height, width_v, height_v (default 17 each), hold_delay, inset x and y
  size self from the pair matching the orientation
  configure dec_button, inc_button and thumb as three-state buttons from the profile
  IF track art absent RETURN false
  work_area <- max(0, long dimension - 2 * short dimension)
  update_scroll_bar()
  unregister thumb from focus navigation
  RETURN true
```

**Notes** — the two orientations read *four* independent numbers rather than a length and a
thickness, so a profile can give the horizontal and vertical bars unrelated proportions. The
default of 17 units for all four is the shipped art's size.

The track length subtracts twice the bar's *short* dimension, on the assumption that a stepper
button is square. The ordinary bar makes the same assumption. It holds for every shipped
profile and would silently mis-size a bar with rectangular arrows.

## `UpdateScrollBar`

**Contract** — the base class's layout with the inset applied: the thumb's maximum length is
reduced by twice the inset on its axis, and only the thumb's long dimension is written — its
short dimension stays whatever the art made it. The track is not repositioned, because it is a
fixed piece of art.

**Notes** — as in the base class, "nowhere to scroll" is expressed by making the thumb
transparent rather than hiding it.

## `OnMouseAction`

**Contract** — this variant runs the drag itself, since a button thumb has no drag behaviour of
its own. A press captures on behalf of the thumb; a release ends it; a move while captured and
while the cursor is inside the thumb's rectangle advances the thumb by the cursor's frame
delta and raises the thumb-moved message on the thumb's behalf. The wheel pages in the
matching direction. Every one of these consumes the event.

```text
FUNCTION on_mouse_action(x, y, action) -> bool
  MATCH action
    wheel_up:   try_scroll_dec(page = true) ; RETURN true
    wheel_down: try_scroll_inc(page = true) ; RETURN true
    left_down:  capture_mouse(thumb)        ; RETURN true
    left_up:    release_mouse(thumb) ; mouse_state <- idle ; RETURN true
    mouse_move:
      over <- cursor inside thumb's absolute rect
      IF thumb is the capturer AND over
        advance thumb along the bar's axis by cursor.frame_delta
        raise THUMB_MOVED as if from the thumb
      IF NOT over THEN release_mouse(thumb)
      RETURN true
  RETURN inherited.on_mouse_action(x, y, action)
```

**Notes** — the difference from the stretchable thumb is severe and is the one behavioural
divergence between the two bars: here the drag survives only while the cursor is *inside the
thumb*, where the ordinary thumb allows a wide margin. Dragging fast enough to outrun the
thumb therefore drops the drag. The source shows the margin test commented out in favour of
the strict one, so the strictness is a deliberate late change with no recorded reason.

Note also that a left press captures unconditionally, before the press hit test decides
whether the press was on a stepper. Capturing the thumb for a press on an arrow is harmless
only because the subsequent move test requires the cursor to be over the thumb.

## `OnMouseDownEx`

**Contract** — the same five-region hit test as the base bar, with one difference: a press in
either page region scrolls by a *single* step rather than a page, and does not latch a
direction. So clicking the track beside the thumb nudges rather than pages.

**Notes** — the latching assignments survive in the source as comments. Without a latch, a held
press in a page region repeats indefinitely and can pass the thumb, which the base bar's latch
prevents.

## `SetPosScrollFromView`

**Contract** — the drag-to-position conversion, identical in form to the base bar's, with one
addition: it re-runs the layout at the end so the thumb is immediately snapped onto the
position it just produced.

**Notes** — the base class routes the same computation through the clamped setter; here the
position field is written and clamped directly. The effect is the same and the duplication is
incidental.

## `ClampByViewRect`

**Contract** — pushes the thumb back inside the track, where the track's ends are the steppers
plus the inset.

## `OnKeyboardAction` / `OnMouseDown` / `OnMouseUp` / `SendMessage` / `Draw`

**Contract** — the base bar's behaviour re-expressed against the button thumb: the held-button
auto-repeat on its own timestamp, the press forwarding, the release clearing the latch, the
steppers' click and press both stepping, and the thumb-moved message driving the conversion
and the owner notification. `Draw` deliberately skips the base *bar's* draw and calls the
plain window draw, since this variant has no separately drawn pieces.

**Notes** — the repeat timestamp is again a single module-wide value rather than a field, and
is a *different* value from the base bar's, so the two bars do not even share a clock. Both
should be per-instance in a rebuild.
