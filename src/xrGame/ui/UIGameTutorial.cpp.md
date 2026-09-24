# src/xrGame/ui/UIGameTutorial.cpp

> The tutorial sequencer: a queue of authored steps played one at a time, which takes the whole input stream, decides per step what to forward to the game underneath, drives the world's pause state, and calls into script at every boundary.

**Needs** — [`UIGameTutorial.h`](UIGameTutorial.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIGameCustom.h`](../UIGameCustom.h.md) · [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrEngine/xr_level_controller.h`](../../xrEngine/xr_level_controller.h.md) · [`xrEngine/XR_IOConsole.h`](../../xrEngine/XR_IOConsole.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`UIGameTutorial.h`](UIGameTutorial.h.md)
**Tier floor** — T3.

## Purpose

Every scripted overlay in the games — the opening tutorial, a full-screen video, a hint that
waits for the player to press a key, the ending slideshow — is one *sequence*: an authored
list of steps read from a single shipped document, played in order.

The sequencer is not a screen. It **does not go on a dialog holder's stack**; it registers
itself directly with the frame loop and the render loop and takes the input receiver, which
lets it draw over anything including the main menu and lets it be running while a screen is
also up. That is the decision the rest of the file follows from.

## State

```text
RECORD Sequencer
  steps           : queue<SequenceStep>   # the remaining steps; the front one is playing
  root            : Window                # the shared canvas every step attaches into
  sound           : optional<Sound>       # one sound for the whole sequence
  name            : text
  previous_input  : InputReceiver         # who had input before; restored on stop
  on_start, on_stop : optional<text>      # script function names
  flags : { pause_on, pause_off, stored_pause_state,
            persistent, play_each_item, active, over_main_menu }
```

**Invariants**

- **Exactly one step is live**: the queue's front. Steps ahead of it have not been started,
  steps behind it have been destroyed.
- The sequencer restores whatever pause state it found. It records the world's pause state at
  start and undoes its own change at stop, in both directions — a sequence that paused an
  unpaused world unpauses it, and a sequence that unpaused a paused world re-pauses it.
- While active, this object *is* the input receiver. Losing that without restoring the
  previous one leaves the game unable to be controlled, which is why release happens in
  destruction rather than in stop.

## Starting a sequence

**Contract** —

```text
FUNCTION start(name) -> bool
  document := load the shipped tutorial document
  IF the document has no steps named `name` THEN report and RETURN false
  register with the frame loop at a very low priority
  read the sequence-level flags: play_each_item, persistent, over_main_menu, render priority
  IF the display is widescreen AND a widescreen root element exists
    THEN define the named colour "tut_gray" as opaque white; build the root from it
    ELSE define "tut_gray" as mid grey;               build the root from the ordinary element
  read the pause policy: "on", "off" or ignore
  read the sequence-wide sound and the two script hooks
  FOR EACH step element
    build a video step or a simple step by its declared type; load it
  register with the render loop at the authored priority
  start the first step that passes its own precondition
  take the input receiver, remembering the previous one
  apply the pause policy
  play the sound; call the start hook
  RETURN true
```

**Notes** — four things here are load-bearing.

*A named colour is redefined at load time.* The layout vocabulary has a closed named-colour
table (chapter 15); this reaches in and redefines one entry before building the root, so that
the same authored document renders its dimming veil opaque on a widescreen display and grey
otherwise. It is a global side effect from a screen's construction, and it is the only
instance of this in the chapter. A rebuild expresses it as two authored colours and picks one.

*A missing sequence is a logged failure, not a fault.* Scripts start sequences by name and
mods remove them; the caller gets false and carries on.

*Render priority is authored per sequence.* That is what lets one sequence draw under another
— the second tutorial slot exists precisely so two can overlap.

*The frame registration priority is a very low constant*, so the sequencer updates after
everything else in the frame. No reason is recoverable for the exact value beyond "last".

## Choosing the next step

**Contract** — a step may carry a **precondition**: the name of a script predicate. Steps
whose predicate is false are **discarded**, not skipped-and-kept, and the search continues
into the queue until one passes or the queue empties.

```text
FUNCTION next_playable() -> optional<Step>
  WHILE steps is not empty
    candidate := steps.front
    IF candidate has no precondition THEN RETURN candidate
    IF script(candidate.precondition)() THEN RETURN candidate
    discard steps.front
  RETURN none
```

**Notes** — the predicate is evaluated **at the moment the step would start**, not at load,
which is what makes a tutorial able to branch on what the player has already done. A missing
script function is fatal rather than false: a typo in a tutorial document must not silently
skip a step.

## The frame

**Contract** —

```text
FUNCTION on_frame()
  IF the application is not focused, or the sequencer is inactive, THEN RETURN
  IF no steps remain THEN stop; RETURN
  IF the front step reports it has finished playing THEN advance
  IF no steps remain THEN stop; RETURN
  update the front step and the shared root
```

**Notes** — the two emptiness tests around the advance are not redundant: advancing can empty
the queue, and the update that follows would otherwise read the front of an empty queue.

Advancing asks the current step's `Stop` whether it *may* stop, and **a step that refuses
stays**. That is the mechanism behind "press the key to continue": the step declares a guard
action and refuses to stop until that action has been seen.

## Stopping

**Contract** — two paths and they differ:

- *Stop the sequence* — if steps remain and the sequence is in "play each item" mode, this
  only advances to the next step; otherwise it force-stops the current step, restores the
  pause state and destroys everything.
- *Destroy* — call the stop hook, stop the sound, unregister from both loops, delete the
  remaining steps and the root, release input, fire the destruction callback, and clear
  whichever of the two global sequencer slots pointed here.

**Notes** — the "play each item" flag turns the escape key from *cancel the whole tutorial*
into *skip this step*, which is why the same key press means different things in different
sequences. It is the only flag that changes what stopping means.

The two **global sequencer slots** are the multiplicity limit: at most two sequences can run
at once, one over the other. Destruction clears whichever slot named it, so a sequence cannot
outlive its slot and a slot cannot point at a destroyed sequence.

## Input routing

**Contract** — while active, the sequencer receives every input event before anything else,
and forwards to the previous receiver **only when the current step does not grab input**.

```text
FUNCTION on_key_press(code)
  step := the front step, if any
  offer the code to the step
  allowed := step has not disabled the action this code is bound to
  is_cancel := code is bound to quit, or to the UI back action in the UI context
  IF allowed AND is_cancel THEN stop; RETURN
  IF is_cancel AND a game screen is up
    close the topmost of: the inventory screen, the PDA screen; else open the main menu
    RETURN
  IF allowed AND the step does not grab input THEN forward to the previous receiver
```

**Notes** — **a step disables actions by name, not by key**, so a tutorial that forbids
movement forbids it under the player's own bindings. The disabled set is consulted before the
cancel test, which means a tutorial can make itself uncancellable by disabling the quit
action.

The cancel branch has a second life as a general "back" handler: when the tutorial is running
over a game screen, the back key closes that screen rather than the tutorial, falling through
to opening the main menu when nothing is up. That is the sequencer standing in for the screen
stack it deliberately does not participate in.

Every other channel — release, hold, movement, wheel, controller — is pure forwarding gated
on the grab flag. Press is the only channel with logic, because press is the only one a step
reacts to.

## Regaining focus

**Contract** — when the application regains focus, every key **currently held** that is bound
to a movement, look, crouch, sprint, lean or fire action is replayed as a fresh press.

**Notes** — the game tracks movement as held state. Losing focus drops the releases; without
this replay the player returns to a game that thinks they let go of everything, and the
tutorial's own "press forward" step would never see the key they are already holding. The
list of replayed actions is exactly the continuous ones — replaying a discrete action such as
*use* would fire it twice.
