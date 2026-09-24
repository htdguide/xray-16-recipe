# src/editors/xrSdkControls/Controls/NumericSpinner/NumericSpinner.cs

> A fractional spin box with a drag handle: press the handle and move the mouse sideways to scrub the value without bound.

**Needs** — [`NumericSpinner.Designer.cs`](NumericSpinner.Designer.cs.md)
**Used by** — [`NumericSpinner.Designer.cs`](NumericSpinner.Designer.cs.md)
**Tier floor** — T3: it hides and warps the pointer, which is a windowing-layer capability but not a device one.

## Purpose

A spin box whose arrows step by a fixed amount is slow when the useful range is wide. This control adds the gesture every digital-content tool has: grab a handle, drag, and the value follows the mouse continuously, with no end of travel because the pointer is recentred as it goes.

## State

```text
RECORD NumericSpinner
  spinner      : the host toolkit's fractional spin box   # authoritative
  handle       : the drag grip
  precision    : real        # value change per pixel of horizontal movement
  dragging     : bool
  anchor_x     : int         # last horizontal position within the handle
  anchor_screen: point       # where to warp the pointer back to
  accumulation : int         # signed count of motion events since the last warp
```

**Invariants** — the value never leaves the spin box's range: the drag clamps at both ends and simply stops moving. The pointer is hidden for exactly the duration of the drag; the hide and show are paired with the press and release.

## The infinite drag

**Contract** — pressing the handle records the anchor, hides the pointer and begins a drag. Each horizontal movement converts the pixel delta into a value delta by `precision`, applies it clamped, and counts toward a recentring step. When the count passes a small threshold, the pointer is warped back to the anchor and the count resets — which is what makes the drag unbounded: the mouse never actually travels far, so it never hits the edge of the screen.

```text
ON handle_press(position)
  dragging = true
  anchor_x = position.x
  anchor_screen = pointer_screen_position()
  hide_pointer()

ON handle_move(position)
  IF NOT dragging OR position.x == anchor_x THEN RETURN
  delta = (position.x - anchor_x) * precision
  value = clamp(value + delta, minimum, maximum)
  IF absolute(accumulation) > 1
    warp_pointer_to(anchor_screen)      # the pointer stays put; the value keeps moving
    accumulation = 0
  accumulation = accumulation + sign(delta)

ON handle_release
  dragging = false
  show_pointer()
```

**Notes** — hiding the pointer is not decoration. Without it the author sees the cursor snap backwards on every warp, which reads as a stutter. Hidden, the gesture feels like an endless wheel.

The recentring threshold is a count of motion *events*, not of pixels, and it is small enough that a warp happens almost every other event. The accumulation counter is signed, so alternating left and right movements cancel and the pointer is left alone — a rebuild could simply warp on every event and lose nothing observable. Why the counter is signed at all, rather than an absolute event count, is not recoverable from the source; it appears to be an attempt at hysteresis that does not survive contact with the threshold of one.

**The anchor is never re-read after a warp.** After the pointer is warped, the next motion event's position is measured from the new pointer location while `anchor_x` still holds the old one, so the first post-warp delta is wrong. In practice the warps are frequent and the error is a pixel; it is nevertheless a real defect a rebuild should not copy — warp, then set the anchor from the warped position.

## `Precision`

**Contract** — value units per pixel dragged. A plain stored number with no validation; zero disables the gesture, negative reverses it.

**Notes** — the source shows this property was originally meant to be the embedded track bar's resolution (the commented-out body says so) before this control lost its track bar. The name survived the change of meaning, which is why it reads oddly.

## Forwarded properties

Value, range, display base, text alignment and decimal places go straight to the embedded spin box. The control stores no value of its own.

## Notes

A suppression flag exists and brackets nothing: the value-changed handler consults it but no write ever sets it. It is dead, left from a two-child arrangement like [`NumericSlider`](../NumericSlider/NumericSlider.cs.md)'s.
