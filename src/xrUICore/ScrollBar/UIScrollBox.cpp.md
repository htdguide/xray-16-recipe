# src/xrUICore/ScrollBar/UIScrollBox.cpp

> The scroll bar's thumb: capture on press, follow the cursor along one axis while captured, report each move to the parent bar.

**Needs** — [`UIScrollBox.h`](UIScrollBox.h.md) · [`Windows/UIFrameLineWnd.h`](../Windows/UIFrameLineWnd.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`UIMessages.h`](../UIMessages.h.md)
**Used by** — [`UIScrollBox.h`](UIScrollBox.h.md)
**Tier floor** — T3: a drag protocol over cursor deltas.

## Purpose

Dragging is separated from the scroll bar because the bar reasons in scroll positions and the
thumb reasons in pixels. The thumb never knows the range, the page size or the content; it
moves by the cursor's frame delta along its own axis and announces that it moved. The bar
converts.

## State

`Stateless.` The drag is carried entirely by the parent's mouse capture and the cursor's
per-frame delta.

## `OnMouseAction`

**Contract** — consumes every mouse action over the thumb. A press takes the parent's mouse
capture; a release drops it. While captured and while the cursor is within a generous margin
of the thumb, the thumb's position is advanced by the cursor's frame delta along the bar's
axis, and the parent is told with a thumb-moved message. Leaving the margin drops the capture.
Always returns claimed.

```text
FUNCTION on_mouse_action(x, y, action) -> bool
  # x, y are relative to the thumb
  near <- x in -512 .. width + 512 AND y in -512 .. height + 512

  IF action IN { left_down, left_double_click }
    parent.capture_mouse(self) ; RETURN true
  IF action == left_up
    parent.release_mouse(self) ; RETURN true

  IF parent.mouse_capturer IS self AND action == mouse_move AND near
    delta <- cursor.frame_delta
    IF horizontal THEN position.x <- position.x + delta.x
    ELSE               position.y <- position.y + delta.y
    message_target.send(self, THUMB_MOVED)

  IF NOT near THEN parent.release_mouse(self)
  RETURN true
```

**Notes** — the 512-unit margin is the load-bearing oddity. In the toolkit's virtual
1024×768 screen that is half the width, so in practice the drag survives the cursor wandering
far off the bar — which is what a scroll bar should do — but not all the way across the
screen. The source marks the constant with a smiley and gives no derivation; treat it as a
tuned "far enough" and copy it.

Movement is taken from the cursor's *delta*, not from the cursor's position relative to a
grab point. The thumb therefore drifts relative to the cursor whenever the bar clamps it, and
re-grips wherever the cursor happens to be. Reproducing this matters: a grab-point
implementation feels different at the ends of the track.

The thumb moves itself before the parent has had any say. The parent's clamp runs when it
handles the moved message, which is why a thumb can be momentarily outside its track within
one frame's handling.
