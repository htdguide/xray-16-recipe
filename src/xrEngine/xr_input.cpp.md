# src/xrEngine/xr_input.cpp

> The input layer — one flat key space covering keyboard, mouse and gamepad, a stack of receivers of which only the top one hears anything, and the per-frame translation of platform events into press/hold/release.

**Needs** — [`xr_input.h`](xr_input.h.md) · [`IInputReceiver.h`](IInputReceiver.h.md) · [`xr_level_controller.h`](xr_level_controller.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`device.h`](device.h.md) · [`GameFont.h`](GameFont.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`xrCore/Text/StringConversion.hpp`](../xrCore/Text/StringConversion.hpp.md) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`xr_input.h`](xr_input.h.md)
**Tier floor** — T2: event pumping and bit sets. It sits directly on the windowing seam, which is where the platform-specific work lives.

## Purpose

Every part of the engine that reacts to a button — the game, the menu, the console, the
debug overlay, a text field — needs input, and only one of them at a time should get it.
This file is the arbitration.

Three decisions define it:

1. **One key space.** Keyboard scancodes, mouse buttons, gamepad buttons and gamepad axes
   are all integers in a single numbering. A binding is one integer. Nothing downstream
   needs to know which device a binding refers to.
2. **A receiver stack, not a broadcast.** Input goes to the top of the stack only. There is
   no "handled" flag and no propagation. Opening the console pushes; closing it pops.
3. **Press, hold, release — derived, not delivered.** The platform gives edges. The engine
   synthesises the *hold* from its own retained state, every frame, for every key that is
   down.

## State

```text
RECORD InputLayer
  keyboard_down   : set of scancodes              # bit per scancode
  mouse_down      : set of mouse buttons
  mouse_axes      : list<int>   # [x, y, scroll_x, scroll_y]
  controller      : ControllerState
  receivers       : list<InputReceiver>           # a stack; only the last one is fed
  controllers     : list<open gamepad handles>
  current_device  : {keyboard_and_mouse, controller}
  text_input_depth: int                           # nesting count, not a flag
  exclusive       : bool                          # relative mouse mode when grabbed
  grabbed         : bool
  key_map_watchers: signal registry

RECORD ControllerState
  left, right          : AxisState    # the two sticks
  trigger_left, right  : AxisState    # the two analogue triggers
  buttons              : set of controller buttons
  gyroscope            : 3 reals
  id                   : optional<int>   # which physical controller is in use

RECORD AxisState
  x, y      : real     # deadzone-corrected, normalised
  magnitude : real     # the corrected length; zero means "not active"
```

Invariants:

- The receiver stack is **never empty**. A permanent bottom receiver is pushed at
  construction, so `receivers.last` is always valid and no call site needs a null check.
  That receiver handles the three things that must work even with nothing else running:
  quit to the main menu, open the console, cycle the debug overlay.
- An axis is "active" exactly when its magnitude is non-zero. That is the sole criterion
  the press/hold/release derivation uses, so the deadzone is what decides whether an axis
  generates events at all.
- The key space's ranges are contiguous and ordered: scancodes first, then mouse buttons,
  then controller buttons, then controller axes, each range beginning one past the previous
  range's end. Every lookup is a range test plus a subtraction. A rebuild is free to choose
  its own numbering, but **the configuration files store bindings by these numbers' names,
  not by the numbers**, so the *names* are frozen and the numbers are not.

## The key space

```text
0 .. scancode_count-1            keyboard, by scancode
scancode_count                   (the "no mouse button" marker)
scancode_count+1 .. +5           mouse buttons 1..5
...                              (the "no controller button" marker)
next 21                          controller buttons, including paddles and touchpad
...                              (the "no controller axis" marker)
next 4                           left stick, right stick, left trigger, right trigger
```

**Notes** — the two sticks are **one key each, not two axes each**. The platform reports
four independent axes; the engine collapses each stick's pair into a single two-dimensional
entry, because a binding to "the left stick" is meaningful and a binding to "the left
stick's horizontal axis" is not. The triggers stay separate because they are independent
controls.

The marker values between the ranges are not dead space; they are what makes each range
test a strict inequality on both sides and they are what an unbound binding stores.

## The per-frame pass

**Contract** — runs once per frame, from the frame signal at high priority so it precedes
everything that reads input. Skipped entirely while the level is precaching or while an
external screenshot tool has the frame. Drains the platform's event queue in three passes:
controller, keyboard, mouse.

```text
FUNCTION on_frame()
  IF the quit chord was seen THEN RETURN      # the process is going away; stop feeding it
  controller_update()
  keyboard_update()
  mouse_update()
```

**Notes** — the order is not arbitrary. Controller first, because it is the pass that may
switch the active device and the other two need to know. Each pass bounds how many events
it takes in one frame (64 keyboard, 256 mouse, 256 controller); a flood beyond that is
carried to the next frame rather than stalling this one.

## Keyboard

```text
FUNCTION keyboard_update()
  events = drain the keyboard event range

  # Pass one: update retained state for the whole batch, so that a query made
  # during this frame's dispatch sees the final state and not a partial one.
  FOR EACH event
    key down (not an auto-repeat): keyboard_down.add(scancode)
    key up:                        keyboard_down.remove(scancode)

  IF F4 is down AND either alt is down THEN
    queue "disconnect" then "quit"; RETURN       # the platform close chord, handled here

  IF there were any events THEN current_device = keyboard_and_mouse

  depth_at_entry = text_input_depth
  FOR EACH event
    key down (not repeat): top_receiver.on_key_press(scancode)
    key up:                top_receiver.on_key_release(scancode)
    text:                  IF text_input_depth == depth_at_entry THEN
                             top_receiver.on_text(composed characters)
    keymap changed:        broadcast to the key-map watchers

  FOR EACH scancode still down
    top_receiver.on_key_hold(scancode)
```

**Invariants and decisions:**

- **State is updated for the entire batch before any of it is dispatched.** A receiver that
  queries "is shift down" during a key-press callback must see this frame's truth, not the
  truth as of the event before it.
- **Auto-repeat is discarded.** The engine derives hold events itself, every frame, at frame
  rate; the platform's repeat rate is a user preference about text editing and has no place
  in a game loop.
- **Text events are suppressed if the text-input nesting depth changed mid-batch.** Opening
  a text field in response to a key press would otherwise deliver that same key's character
  into the field that the key opened. The source is candid that this heuristic is not
  exactly right — it detects "the target changed" by proxy — and a rebuild should instead
  tag each text event with the field that was focused when it was generated.
- The close chord is handled here rather than left to the platform, because the engine must
  disconnect from a server and shut down cleanly first. The two actions are *queued* on the
  event queue rather than performed, so they run at a safe point in the frame.

## Mouse

```text
FUNCTION mouse_update()
  scroll accumulators = 0
  previous = mouse_down
  events = drain the mouse event range

  FOR EACH event
    motion: accumulate the relative delta; record the absolute position
    button down/up: remap the button index, update mouse_down,
                    dispatch press or release
    wheel:  accumulate both axes, in both precise and integer form

  FOR EACH button that is down now AND was down before
    top_receiver.on_mouse_hold(button)

  IF there was motion THEN
    IF the accumulated delta is non-zero THEN top_receiver.on_mouse_move(delta)
    IF the accumulated scroll is non-zero THEN top_receiver.on_mouse_wheel(scroll)
```

**Invariants** — the *middle* and *right* buttons are swapped relative to the platform's
numbering. The engine's numbering has right as button two; the platform's has middle there.
Bindings and shipped configuration use the engine's, so the remap is frozen.

Motion deltas are accumulated across every event in the frame and delivered once. Delivering
each motion event separately would make mouse look depend on the platform's event rate
rather than on the distance moved.

The scroll is accumulated twice, as a precise fractional value for the callback and as an
integer for the pollable state. Precision-capable input devices (touchpads, high-resolution
wheels) produce fractional scroll, and the integer form exists for callers that only want
clicks.

## Controller

```text
FUNCTION controller_update()
  drain the device add/remove/remap range:
    added:   open it; enable its gyroscope if sensors are on
    removed: close and forget it
    remapped: consumed and ignored

  IF no controllers THEN RETURN

  IF there are pending controller input events THEN current_device = controller
  ELSE IF current_device is not controller THEN RETURN   # don't process stale state

  previous = controller state
  drain the controller input range:
    axis motion: record the raw value; note which axes moved; adopt this controller's id
    button down/up: update the button set; dispatch press or release with a
                    full-scale or zero axis state
    gyroscope: IF from the active controller, negate all three components and store;
               dispatch an attitude change

  FOR EACH button down now AND before
    dispatch hold

  apply the deadzone transform to each moved stick and trigger
  FOR EACH of the four axes
    active now and before -> hold
    active now only       -> press
    active before only    -> release
```

**Invariants and decisions:**

- **Buttons and axes are dispatched through the same press/hold/release calls**, with a
  button reported as an axis state of full magnitude. That is what lets a binding be to
  either without the receiver caring: a receiver asks "how far", and a button answers "all
  the way".
- **Only the most recently used controller is listened to.** Multiple gamepads may be open
  (for hot-plug), but the active id is adopted from whichever produced the last event, and
  gyroscope data from any other is dropped. The game is single-player at this layer.
- **Gyroscope axes are all negated** when stored. That is the conversion between the
  platform's sensor frame and the engine's, and the first two are also swapped. Neither is
  derivable from anything in this file; a rebuild must determine it empirically for its own
  sensor source.
- The device switch flushes pending sensor events, so that turning the gamepad on does not
  deliver a burst of accumulated attitude.

### The stick deadzone

```text
FUNCTION apply_stick_deadzone(raw) -> AxisState
  magnitude = length(raw)
  IF magnitude <= inner OR inner >= full_scale THEN RETURN inactive
  direction = raw / magnitude
  magnitude = min(magnitude, outer)
  normalised = (magnitude - inner) / (outer - inner)
  RETURN (direction * normalised, normalised)
```

**Invariants** — this is a *radial* deadzone with both an inner and an outer edge, and it
**rescales** rather than clipping: a stick just past the inner edge produces a magnitude
just above zero, and one at the outer edge produces exactly one. Clipping instead would
make the stick jump from nothing to fifteen percent the moment it left the deadzone, which
is the single most common failure of naive gamepad handling.

The outer deadzone exists because physical sticks do not reach full deflection in the
diagonals; treating 96% as full makes the diagonals reachable.

The direction is preserved exactly, so the deadzone never changes which way the stick
points — only how far.

Triggers get a far simpler treatment: a plain scale to 0..1 with no deadzone at all. That
is a gap, not a decision — a trigger with rest drift will register as permanently pressed.

## The receiver stack

**Contract** — `capture` pushes a receiver and `release` removes one. The outgoing top is
told it lost focus and the incoming top is told it gained it. Releasing a receiver that is
not on top removes it from the middle **without** any focus notification, because the top
never changed.

```text
FUNCTION capture(receiver)
  IF the stack is non-empty THEN stack.last.on_deactivate()
  push receiver
  receiver.on_activate()
  controller state = cleared            # a new receiver starts from neutral

FUNCTION release(receiver)
  IF receiver is the top THEN
    receiver.on_deactivate()
    pop
    stack.last.on_activate()
  ELSE
    remove the topmost occurrence of receiver from the middle
```

**Invariants** — capturing clears the controller state. Without it, a receiver pushed while
a stick is deflected would receive a *release* for an axis it never saw pressed, or worse,
inherit a held button. The keyboard and mouse state are deliberately *not* cleared, because
modifier keys must survive a capture (a receiver pushed while shift is held should see
shift held).

Removal from the middle searches from the top down and removes the first match, so a
receiver pushed twice loses its more recent entry.

## Focus changes

**Contract** — on losing or gaining window focus, the top receiver is told, and **all three
device states are cleared**. Every key is considered released.

**Notes** — this is not tidiness, it is correctness: keys released while the window was not
focused generate no release event, so a key held during an alt-tab would otherwise stay
down forever. The corollary is that a key genuinely held across a focus change reads as
released and will not report held again until it is physically re-pressed.

## Input grabbing

**Contract** — grabbing hides the cursor, confines it to the window, and — in exclusive mode
— switches the mouse to relative mode, where the platform reports deltas and the pointer
does not move. Releasing undoes all three.

**Notes** — exclusive mode is the difference between "mouse look" and "a cursor in a menu",
and it is a *user setting* rather than a mode the engine picks, because relative mouse mode
interacts badly with some window managers and some accessibility tools. Changing it
re-grabs, so the change takes effect immediately.

## Text input

**Contract** — a nesting counter, not a flag. The first enable starts text composition; the
last disable stops it. Both ends flush pending composition events.

**Notes** — the counter exists because several things can want text input at once (a console
that is open behind a debug overlay's text field), and the naive flag would have whichever
closed first turn it off for both. The counter is clamped at zero so an unbalanced disable
degrades rather than corrupting.

The flush on both transitions discards half-composed input, which is what you want when the
focus moves: a partially-composed character belongs to the field that was focused.

## `iGetAsyncKeyState` and the polled state

**Contract** — answers "is this key down right now" for any point in the key space, by
range-testing the integer and consulting the matching state. An axis answers by whether its
magnitude is non-zero. An unrecognised value answers no rather than failing.

**Notes** — this is the *polled* half of the input layer and it coexists with the callback
half. Both are needed: continuous movement is naturally polled, discrete actions are
naturally event-driven.

## `GetKeyName`

**Contract** — produces a human-readable name for a key, for display in the bindings screen
and in menus. Only keyboard keys have names; mouse and controller entries fall through to a
stub that always fails.

**Notes** — the platform's key names are UTF-8 and the engine's text is in a single-byte
codepage (see the preface), so the name is converted through the current locale. That
conversion is lossy for names the codepage cannot represent, and the failure mode is a
blank binding label.

The unimplemented half means the bindings screen has no name for a gamepad button. That is
a real gap; see [`xr_level_controller.cpp`](xr_level_controller.cpp.md), which carries its
own name table for exactly these.

## `Feedback`

**Contract** — starts a rumble effect on the active controller, in either the main motors
or the trigger motors, with two intensities in 0..1 and a duration in seconds. Each call
cancels the previous effect; zero intensity stops it. Does nothing when no controller is
active.

**Notes** — the intensities are scaled to the platform's full 16-bit range and the duration
to milliseconds, with a negative duration meaning zero rather than forever.

## `DumpStatistics`

**Contract** — reports the time the input pass took, on the debug overlay.

## Notes

The tuning values — mouse sensitivity and inversion, stick sensitivities per axis, the three
deadzones, gyroscope sensitivity and deadzone, and the cursor auto-hide delay — are
process-wide and are registered as console variables elsewhere. They are declared here
because this is where they are read, and their defaults are here: separate sensitivities
for the stick's two axes (0.12 horizontal, 0.7 vertical) is the notable one, and it is a
feel decision with no derivation.

The mouse-position and mouse-warp calls have a global-coordinates variant that not every
platform supports. Where it is unsupported the call silently falls back to window
coordinates **and reports failure**, so the caller can tell the difference between "done"
and "approximated". That two-valued result is the right shape for a capability that may or
may not exist.
