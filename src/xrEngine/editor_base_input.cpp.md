# src/xrEngine/editor_base_input.cpp

> The overlay's platform backend: translating the engine's input events into the toolkit's, owning the cursor and the text-input mode, and persisting each tool's settings.

**Needs** — [`editor_base.h`](editor_base.h.md) · [`editor_helper.h`](editor_helper.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`xr_input.h`](xr_input.h.md) · [`device.h`](device.h.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`editor_base.cpp`](editor_base.cpp.md) · [`editor_base.h`](editor_base.h.md)
**Tier floor** — T2: event translation and a per-frame pointer query; no data layout or budget.

## Purpose

The overlay seam says the engine supplies the platform half of the toolkit and the toolkit
supplies the widgets. This file is that platform half: it converts the engine's own input
events into the toolkit's event queue, decides what the mouse cursor looks like, turns the
platform's text-input mode on and off, tracks which window the pointer is over when tools
are torn off into their own windows, and implements the settings file each tool reads and
writes.

The reason this is the engine's job rather than a library's is that the engine has already
taken the input: the pointer is grabbed and keys are being delivered to the game as
*actions*, not scancodes. Nothing generic could hook in below that.

## Backend state

```text
RECORD Backend
  pointer_window      : window id    # which window the pointer most recently entered
  left_at_frame       : int          # the frame after the pointer left a window; 0 = not left
  can_report_hover    : bool         # this platform can say which window is hovered
  text_input_enabled  : bool         # mirrors what we last told the platform
```

## `InitBackend`

**Contract** — declares what this backend can do, decides whether tools may be torn off into
their own platform windows, and installs the settings handler.

```text
FUNCTION init_backend(editor)
  declare: we supply gamepad input, mouse cursors, and can move the pointer
  IF the platform's video driver is one of a known-good list
    declare that tools may occupy their own platform windows
    IF not on a platform where hover reporting is unreliable
      can_report_hover = true
  install a settings handler named for this engine
```

**Notes** — the multi-window capability is decided by an **allow-list of video drivers**, not
a feature test, because the capability being probed — a global pointer position across
windows — has no query. The list names the drivers known to have it; two common ones do not,
and the source says a deny-list was considered and rejected. That is the right call for a
capability whose absence manifests as a tool window that cannot be dragged.

One platform is excluded from *hover reporting* specifically while keeping multi-window
support, which is a narrower defect on that platform. Both exclusions are empirical and a
rebuild on a different windowing library will have its own.

## The settings handler

**Contract** — teaches the toolkit's own settings file how to carry per-tool sections. Five
hooks: forget everything, begin a named section by finding the tool that owns it, deliver one
line to that tool, tell every tool the file is fully read, and write every tool's section
back.

```text
clear all      -> every tool forgets its settings
open section   -> find the tool whose name matches; clear it first (the entry may be
                  recycled) and return it as the section's target; unknown names are dropped
read line      -> hand the line to that tool
apply all      -> tell every tool the file is complete
write all      -> sum every tool's size estimate, reserve once, then write
                  "[engine][tool name]" and let the tool append its own lines
```

**Notes** — clearing a section on open rather than on close is what makes a settings file
that names the same tool twice end with the second copy rather than a merge of both.

An unknown section name is silently dropped, which is what lets a settings file outlive the
tool that wrote it.

## `UpdateMouseData`

**Contract** — run once per frame while the overlay holds input. Resolves the pointer's
position, which window it is over, and whether the toolkit is allowed to move it.

```text
FUNCTION update_mouse_data(editor)
  IF the pointer has left a window AND a button is now down
    forget the window and report the pointer as being nowhere      # see note
  declare hover reporting on or off: on, unless a drag payload is in flight
  capture the pointer at the platform level while any button is down
  focused = the keyboard-focused window is ours, or is one of the toolkit's
  IF focused AND the toolkit wants to move the pointer
    move it
  IF hover reporting is on
    tell the toolkit which of its viewports the pointer is over
```

**Notes** — the first rule reads oddly and is a real fix: a press that arrives *after* the
pointer has left every window is a press with no target, and reporting the pointer as being
nowhere is what stops the toolkit treating it as a click on whatever was last hovered. The
one-frame lag in the leave bookkeeping is why this is expressed in frame numbers.

Hover reporting is suppressed **while a drag is in flight**, because during a drag the
toolkit needs the pointer's motion, not a changing hover target; reporting both makes a drag
across a window boundary drop its payload. The source notes that testing for a held button
instead would be stricter and noisier, and chose the payload test.

Capturing the pointer while a button is down is what makes a drag that leaves the window
keep delivering motion — the same reason any windowing toolkit does it.

## `UpdateMouseCursor`

**Contract** — maps the toolkit's cursor request to a platform system cursor and shows it;
hides the platform cursor entirely when the toolkit draws its own or wants none. Skips
everything when the toolkit has been told not to touch the cursor.

**Notes** — nine cursors, all standard. The mapping is a translation table and the only
decision in it is that an unrecognized request asserts rather than falls back silently.

## `UpdateTextInput`

**Contract** — keeps the platform's text-input mode equal to what the toolkit wants, changing
it only on a transition. A force-disable path turns it off regardless, used when the overlay
gives input back.

**Notes** — mirroring the state rather than setting it every frame matters because on some
platforms enabling text input raises an on-screen keyboard, and re-enabling it every frame
would re-raise it.

## `IR_OnActivate` / `IR_OnDeactivate`

**Contract** — the two transitions that matter. Taking input **releases the pointer grab**, so
the player's mouse becomes a free cursor; giving it back **re-grabs**. Each also sets the
platform hint that decides whether the click which focuses a window is also delivered to
that window.

**Notes** — the click-through hint is set *on* while the overlay is active and *off*
otherwise, so that the first click into a tool window both focuses it and presses the widget
under it. In a game, the same behaviour would fire a weapon on the click that returns focus
to the window, which is why it is not left on.

Giving input back also force-disables text input, because the toolkit will not get another
frame in which to ask for it to be turned off.

## Keyboard translation

**Contract** — the overlay claims four game *actions* before anything reaches the toolkit, and
each claim is conditional on what has focus. Everything else is translated to a toolkit key
and pushed into its queue.

```text
FUNCTION on_key_press(editor, key)
  CASE the action bound to this key IS
    editor  -> cycle the visibility state ; consume
    console -> IF no overlay window has focus: toggle the engine console ; consume
    scores  -> IF no overlay window has focus: toggle the statistics overlay ; consume
    quit    -> IF the toolkit wants text input, or its item picker is armed: pass through
               ELSE IF an overlay window has focus: drop that focus ; consume
               ELSE: hide the overlay ; consume
  push the modifier state for control, shift, alt and the platform key
  translate the key and push it, if it has a translation
```

**Notes** — the quit key's three-level behaviour is the decision worth copying. Escape means
"back out of one thing", and which thing depends on what is in front: a text field being
edited keeps it, a focused window loses focus, and only an overlay with nothing focused
closes. Each press backs out exactly one level, which is the only behaviour that does not
lose a person's work.

Claiming *actions* rather than scancodes means these follow the player's key bindings, the
same rule the console uses.

Releasing a modifier only clears it **if the other side of the same modifier is not still
held**, which is what keeps Ctrl held through a left-to-right hand change.

## Mouse, controller and text translation

**Contract** — presses and releases are forwarded; holds are not, because the toolkit derives
them. Mouse motion arrives as a *delta* and the toolkit needs an *absolute* position, so the
handler discards the delta and asks the input layer for the current absolute position
instead. Wheel motion is forwarded on both axes. Text input is forwarded only when the
toolkit says it wants it.

Controller buttons map to the toolkit's gamepad keys directly; the two triggers are
forwarded as analogue values so a partial pull reads as a partial press. The two sticks are
**not** forwarded — the code for it is present and disabled — so the overlay is navigable by
d-pad and buttons but not by stick.

**Notes** — the delta-to-absolute conversion is the seam's shape showing through: the engine
grabs the pointer and receives relative motion because that is what a first-person camera
needs, and an interface toolkit needs the opposite. Asking the input layer for the absolute
position each time is the cheap correct answer.

The controller attitude (a gyroscope, on the pads that have one) is received and discarded.
