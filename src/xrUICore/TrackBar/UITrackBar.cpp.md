# src/xrUICore/TrackBar/UITrackBar.cpp

> The options-screen slider: a value quantized to a step, mapped onto a knob's travel along a stretchable bar, driven by dragging, the wheel, the arrow keys or a stick.

**Needs** — [`UITrackBar.h`](UITrackBar.h.md) · [`InteractiveBackground/UI_IB_Static.h`](../InteractiveBackground/UI_IB_Static.h.md) · [`Buttons/UI3tButton.h`](../Buttons/UI3tButton.h.md) · [`Static/UIStatic.h`](../Static/UIStatic.h.md) · [`Options/UIOptionsItem.h`](../Options/UIOptionsItem.h.md) · [`Cursor/UICursor.h`](../Cursor/UICursor.h.md) · [`XML/UITextureMaster.h`](../XML/UITextureMaster.h.md) · [`ui_focus.h`](../ui_focus.h.md) · [`UIMessages.h`](../UIMessages.h.md) · [`xrEngine/xr_input.h`](../../xrEngine/xr_input.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Data: User settings](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`UITrackBar.h`](UITrackBar.h.md)
**Tier floor** — T3: a linear map with quantization, plus four input paths.

## Purpose

Every continuous setting in the options screens — volume, gamma, sensitivity, view distance —
is this widget. It is also used as a two-position switch, because an integer track bar with a
range of zero to one is a toggle, and the options screens use it that way rather than
introducing a second control.

Three things make it more than "position equals value times width". The value is *quantized*
to a step, so dragging snaps rather than glides. The bar may be *inverted*, so that the same
widget can present a setting that decreases to the right. And the bar itself is an
interactive background — its art changes with the enabled state — while the knob is a separate
button drawn on top of it.

## State

```text
RECORD TrackBar EXTENDS InteractiveFrameLine, OptionsItem
  knob        : Button         # drawn, never independently interactive
  label       : Static         # optional numeric readout of the value
  label_format: text           # how to render the value; defaults by type
  is_float    : bool           # true: the value is real; false: integer
  inverted    : bool           # left is the maximum rather than the minimum
  dragging    : bool
  bounds_preset : bool         # the caller supplied the range; do not read it from the setting

  value, minimum, maximum, step, backup   # all real or all integer, per is_float
```

**Invariants**

- The value lies in `minimum .. maximum` at all times; every path clamps.
- The knob's position is always derived from the value — it is never the source of truth. Any
  code that moves the knob must go through the value.
- The numeric readout is drawn only when the label is *enabled*; enabling it is how a layout
  asks for a readout at all. The label is disabled at construction.
- The knob is removed from focus navigation and the *bar* is registered in its place, because
  the bar owns every input path; a focusable knob would split the two.

**Notes on the value's representation** — the five numeric fields are held in one storage slot
reinterpreted as either reals or integers according to the type flag. That is an artifact of
C++ and a hazard: reading the wrong member yields nonsense rather than an error, and the type
flag is the only guard. In a rebuild the value is one tagged number, or two separate widget
types; the flag must survive in some form because the same widget must save to both an
integer and a real setting.

## `InitTrackBar`

**Contract** — builds the bar's art and the knob's, probing two generations of texture names
for each and falling back to the older set. Sizes the knob from its texture, corrected for the
screen's aspect, and starts in the enabled state.

```text
FUNCTION init_track_bar(position, size)
  build the interactive background at position and size
  IF the newer bar art exists
    use it for both the enabled and the disabled state
  ELSE
    use the older enabled and disabled art

  (w, h) <- size of the newer knob art, else of the older knob art
  w <- w * horizontal_aspect_correction
  knob.init at (0, 0) sized (w, h) with whichever knob texture exists
  state <- enabled
```

**Notes** — the knob's *width* is scaled by the aspect correction and its height is not. The
toolkit's virtual screen is 4:3; on a wider display, horizontal distances are corrected so
round things stay round, and the knob's travel — a horizontal distance — must be corrected
with it or the knob would stop short of the bar's end. The height needs no correction because
vertical units are the unscaled ones.

The disabled art is only loaded when the *newer* bar art was found; the older path loads both
states independently and tolerates either being absent. A rebuild needs both states for the
greyed-out look on unavailable settings.

## `UpdatePos`

**Contract** — the forward map: value to knob position, plus the readout. The single writer of
the knob's position.

```text
FUNCTION update_pos()
  travel <- bar width - knob width
  x <- (value - minimum) * travel / (maximum - minimum)
  IF inverted THEN x <- travel - x
  knob.x <- x

  IF label enabled
    render value into label using label_format,
      defaulting to one decimal place for reals and plain digits for integers
```

**Notes** — the knob's *left edge* travels from zero to `travel`, so the knob is inset by its
own width at the maximum rather than centred on the bar's end. Every shipped bar is drawn on
this assumption.

The readout goes through the string table, so a format string may name a localized template —
which is how a setting reads "384 MB" in one language and something else in another.

## `UpdatePosRelativeToMouse`

**Contract** — the inverse map: cursor to value, quantized to the step. Notifies the owner
only if the value actually changed, and always refreshes the knob and marks the setting dirty.

```text
FUNCTION update_pos_relative_to_mouse()
  before <- value
  travel <- bar width - knob width
  p <- cursor x within the bar
  IF inverted THEN p <- bar width - p
  clamp p into (knob width / 2) .. (bar width - knob width / 2)

  raw <- (maximum - minimum) * (p - knob width / 2) / travel + minimum

  # snap to the nearest multiple of the step above the minimum
  d <- raw - minimum
  q <- step * floor(d / step)
  IF d - q > step / 2 THEN q <- q + step
  value <- clamp(minimum + q, minimum, maximum)

  IF value != before THEN message_target.send(self, VALUE_CHANGED)
  update_pos()
  mark the setting as changed
```

**Notes** — the cursor is clamped to *half a knob* in from each end before the map, so the
cursor is treated as grabbing the knob's centre while the map's origin is the knob's left
edge. That half-knob offset appears in both the clamp and the numerator and must be kept
together; dropping one of them shifts the whole scale by half a knob.

The snap rounds to nearest — floor to a multiple, then add one if more than half a step past
it — rather than truncating, so dragging feels centred on each detent rather than lagging
behind it.

The quantization is applied to the *distance above the minimum*, not to the value, so a range
whose minimum is not a multiple of the step still has its detents aligned to the minimum. That
is the right choice and is why a gamma setting from 0.5 to 2.0 in steps of 0.1 lands on 0.5,
0.6, … rather than on 0.5, 0.6 from a grid anchored at zero.

The change notification uses the *button clicked* message, not a slider-specific one. The
whole toolkit funnels "this control was operated" through that one message, which is why an
options screen can subscribe to one message for every control on it.

## `OnMouseAction`

**Contract** — a left press inside the bar starts a drag and immediately jumps the value to the
cursor, so clicking anywhere on the bar moves the knob there. Moving while dragging, with the
button still physically down, updates continuously. A release ends it. The wheel steps one
detent per notch. All five are consumed.

**Notes** — the drag re-checks the physical button state on every move rather than trusting
the event stream, which is what lets a drag survive the pointer leaving the widget and lets a
release outside it be noticed. The per-frame update carries the same check as a backstop, for
the case where no further move arrives.

Wheel up steps *left* — toward the minimum on a normal bar. On an inverted bar the direction
flips with everything else, because inversion is applied inside the step, not at the input.

## `StepLeft` / `StepRight`

**Contract** — move the value by one step in the named direction, honouring inversion, clamp,
notify the owner, refresh the knob and mark the setting changed.

**Notes** — the direction names are *screen* directions: on an inverted bar, stepping left
increases the value. Keeping them screen-relative is what lets the keyboard and gamepad paths
stay ignorant of inversion.

## `OnKeyboardAction` and `OnControllerAction`

**Contract** — the pointer-free paths, both gated on the pointer being over the bar. A press
bound to the UI's move-left or move-right action steps one detent. A stick deflected mostly
horizontally — more than half on the horizontal axis and less than a fifth on the vertical —
steps in the deflection's direction. Either way the cursor is warped onto the knob afterwards.

**Notes** — the deadzone shape is the load-bearing part: the horizontal threshold is generous
and the vertical one is tight, so a stick pushed diagonally moves the *focus* between controls
rather than adjusting this one. Without the vertical bound, a diagonal push would do both.

The cursor warp is what keeps hover, focus and the pointer in agreement on a gamepad; without
it the pointer would sit still while the knob moved out from under it and the next move event
would be ignored.

Both handlers require the pointer to be over the widget, so a gamepad user must have moved the
cursor onto the bar — which the focus system does for them.

## The options-item protocol

**Contract** — reading pulls the value and, unless the caller already supplied bounds, the
range too; then refreshes the knob. Saving writes the value with the type's own writer.
Backing up and undoing copy the value. The change test compares exactly for integers and
approximately for reals.

**Notes** — the preset-bounds flag exists because some ranges are computed at runtime — the
texture-memory slider's maximum depends on the hardware — and must not be overwritten by
whatever the settings file happens to contain. When the flag is set, the setting's own bounds
are read into throwaway variables and discarded, which is a clumsy way of saying "read the
value only".

## `SetOptIBounds` / `SetOptFBounds` / `SetStep` / `SetInvert` / `SetType` / `SetBoundReady`

**Contract** — configuration. Setting bounds re-clamps the current value and marks the setting
changed if the clamp moved it, so a narrowed range cannot leave an out-of-range value behind.
The step setter writes whichever representation the type flag selects, truncating for
integers.

## `GetCheck` / `SetCheck`

**Contract** — the toggle face of the widget, valid only on an integer bar: reading asks
whether the value is non-zero, writing snaps it to the maximum or the minimum. This is how a
plain on/off setting is expressed without a second control type.

## `OnMessage`

**Contract** — responds to one named command, "set to default", by centring the value in its
range and refreshing. This is the options screen's reset button reaching every control by
name.

**Notes** — "default" being the *midpoint* is a simplification, not a stored default: a
setting whose shipped default is not the middle of its range will not return to it.

## `Draw` / `Update` / `Enable`

**Contract** — drawing paints the bar's stateful background, then the knob, then the readout,
in that order; the knob and the label are drawn explicitly rather than by the child walk, so
their z-order is fixed. Enabling swaps the background's state art and the knob's enabled flag
together. The per-frame update drops a drag whose button is no longer held.
