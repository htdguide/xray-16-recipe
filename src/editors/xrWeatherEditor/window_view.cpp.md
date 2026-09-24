# src/editors/xrWeatherEditor/window_view.cpp

> The engine's window inside the editor: where the frame is pumped, where input is claimed, and where two gestures edit a value by pointing at the world.

**Needs** — [`window_view.h`](window_view.h.md) · [`window_ide.h`](window_ide.h.md) · [`ide_impl.hpp`](ide_impl.hpp.md) · [`property_container.hpp`](property_container.hpp.md) · [`property_color_base.hpp`](property_color_base.hpp.md) · [`resource.h`](resource.h.md) · [`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`window_view.h`](window_view.h.md)
**Tier floor** — T1: it clips and hides the system cursor, reads a pixel back from the rendered surface, and drives the engine's frame.

## Purpose

The busiest file on the editor side, and the one that answers the interop question the
whole tool is built around: **when does the engine get a frame, and who owns the mouse?**

## State

See [`window_view.h`](window_view.h.md).

## The frame pump

```text
ON paint
  IF editor.is_idling()
    RETURN                  # the application's own idle loop is already pumping
  engine.on_idle()          # otherwise, this paint is the frame

FUNCTION on_idle()
  check_cursor()            # called from the application's idle loop, every frame
```

**Contract** — the engine advances one frame per paint, unless the application's idle loop
is running, in which case that loop pumps instead and the paint does nothing.

**Notes** — **This is the load-bearing interop decision of the whole editor.** A fixed-rate
engine inside an event-driven application has no loop of its own, so its frame must be
hung off something the application does often. Two candidates exist and both are used: the
application's idle handler, which runs whenever no message is pending, and the panel's
paint, which runs when the window is invalidated. The idle path is the normal one; the
paint path covers the case where the idle loop is not running — a modal dialog is open, or
the window is being dragged — so the view keeps updating instead of freezing.

The guard is what keeps them from both firing: **exactly one of the two pumps at a time.**
A rebuild that inverts the arrangement — engine owns the loop, panels are drawn by it —
deletes both paths, which is the recommendation in
[`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md).

## Edit mode: who owns the mouse

```text
ON edit_toggle_clicked
  edit_toggle.checked = NOT edit_toggle.checked
  pause_toggle.enabled = edit_toggle.checked
  IF edit_toggle.checked                 # editing: the pointer belongs to the interface
    engine.pause(pause_toggle.checked)
    engine.capture_input(true)
    release the pointer clip
    show the pointer
  ELSE                                   # flying: the pointer belongs to the engine
    engine.pause(false)
    engine.capture_input(false)
    clip the pointer to the surface and hide it
```

**Contract** — the toggle switches the pointer between the interface and the engine's
camera. Editing also allows pausing; flying always runs.

**Invariants** — hiding and showing the pointer uses a counter the platform maintains, so
each call loops until the count crosses zero rather than toggling once. Incidental, and a
symptom of a shared global; a rebuild sets a state.

**Notes** — The naming reads backwards until you see it from the author's side: **checked
means "I am editing the grid", unchecked means "I am flying the camera".** In edit mode the
engine still has an input claim — it is the *pointer clip* that is released — so keyboard
shortcuts still reach the camera. That is deliberate: an author nudges the camera with the
keyboard while typing values into a grid.

Pausing is offered only while editing because pausing while flying would freeze the thing
you are flying through.

## Cursor clipping

```text
FUNCTION reclip_cursor()
  IF edit_toggle.checked THEN RETURN          # editing: the pointer is free
  hide the pointer
  clip it to the surface, excluding the toolbar's height at the top

ON size_changed      : engine.on_resize(); reclip_cursor()
ON position_changed  : IF parented AND loaded THEN reclip_cursor()
ON activated         : show the pointer; reclip_cursor()
ON deactivated       : release the clip; show the pointer;
                       IF NOT editing THEN switch to edit mode
ON double_click      : IF editing THEN switch to fly mode
ON alt+enter released: IF editing THEN switch to fly mode
```

**Contract** — while flying, the pointer is confined to the rendered surface so the camera
can be turned without the pointer escaping onto another monitor.

**Invariants** — the clip excludes the toolbar, which is inside the panel but not inside
the rendered surface. Losing focus always releases the clip and returns to edit mode, so a
tool that loses focus can never hold the pointer hostage.

**Notes** — Releasing the clip on deactivation is the safety property that makes this
design acceptable at all. Every other handler here exists to keep the clip in step with a
window that can be docked, floated, moved and resized under it.

Double-click and Alt+Enter both leave fly mode, which gives the author two ways out
without reaching for the toolbar. Note the asymmetry: they only leave, never enter.

## Drag-to-increment

```text
ON pointer_down(middle)
  previous_pointer = pointer location

ON pointer_moved(middle held)
  IF horizontal movement is zero THEN RETURN
  IF no property_grid THEN RETURN
  property_grid.refresh()
  row = the grid's selected row, ELSE RETURN
  value = the model value behind that row
  IF value cannot be incremented THEN RETURN
  value.increment(BY horizontal movement in pixels)
  previous_pointer = pointer location
```

**Contract** — dragging with the middle button over the rendered surface nudges the
*selected grid row's* value, one unit per pixel of horizontal movement, and the result is
visible in the view immediately.

**Notes** — **This is the editor's best idea.** An author selects a fog density, points at
the world, and drags — watching the scene rather than the number. It works because the
grid and the world are the same object: there is no apply step. A rebuild should keep it,
and should note the two pieces it needs: a *current row* the view knows about, and a
value type that can say "I can be nudged" and by how much.

The row is reached by unwrapping the grid's row descriptor back to the model value, which
is the boundary crossing described in
[`property_vec3f_reference.cpp`](property_vec3f_reference.cpp.md) seen from the other
side. A value that cannot be nudged — a text, a name — is ignored silently.

One unit per pixel is unscaled, so dragging a bounded zero-to-one density crosses its whole
range in one pixel. A rebuild should scale the step by the row's range.

## Sample-a-colour

```text
ON pointer_clicked(left, with Alt held)
  IF no property_grid THEN RETURN
  row = the grid's selected row, ELSE RETURN
  value = the model value behind that row
  IF value is not a colour THEN RETURN
  pixel = read the rendered surface AT the click position
  value = (red, green, blue) each scaled FROM 0..255 TO 0..1
  property_grid.refresh()

FUNCTION pick_color_cursor() -> bool        # the same test, without acting
FUNCTION check_cursor()                     # run every idle frame
  set the surface's cursor TO (eyedropper IF pick_color_cursor() ELSE default)
```

**Contract** — holding Alt and clicking the rendered world sets the selected colour row to
the pixel under the pointer. The cursor changes to an eyedropper whenever that gesture
would work, re-tested every frame.

**Notes** — The counterpart of drag-to-increment, and the same idea: **author by pointing
at the result.** Sampling the fog colour from the fog, or the sky colour from the sky, is
how these values are actually chosen.

Reading a pixel back from the presented surface is the mechanism, and it is a genuine
constraint on a rebuild: it requires the rendered image to be *readable*, which not every
presentation path allows. The alternative — asking the engine what colour it computed at
that point — is better and is not what this does.

The colour arrives from the platform with its red and blue components in the opposite order
to the one the engine uses, and the extraction here re-orders them. That byte order is a
property of the pixel read, not of the engine's colour, and a rebuild inherits whatever its
own read gives it.

**Re-testing the whole condition every frame to decide a cursor shape** is wasteful — it
walks the grid's selection and unwraps a row sixty times a second to answer a question that
changes on selection and on the Alt key. A rebuild recomputes it on those two events.

## `on_load_finished` and `pause`

```text
FUNCTION on_load_finished()
  IF already loaded THEN RETURN
  loaded = true
  switch to edit mode
  toggle pause

FUNCTION pause()
  IF editing THEN RETURN
  switch to edit mode
```

**Contract** — when the level finishes loading the tool lands in edit mode, paused. When
the engine asks the editor to pause, the tool leaves fly mode so the author gets the
pointer back.

**Notes** — Starting paused is right for a weather tool: the author arrives at a fixed
moment in the day rather than watching it drift while they orient themselves.
