# src/xrGame/ActorEffector_script.cpp

> The one piece of the effector system that has to reach into the script virtual machine: telling a script when its camera animation has finished.

**Needs** — [`ActorEffector.h`](ActorEffector.h.md) · [`xrEngine/ObjectAnimator.h`](../xrEngine/ObjectAnimator.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one callback dispatch

## Purpose

A script that plays a cutscene camera move needs to know when the move is over, so it can
give control back to the player. This file is the two methods that make that work. It is a
separate file only because it must be compiled against the script-engine headers, which
the rest of the effector code is not; a rebuild should merge it into
[`ActorEffector.cpp`](ActorEffector.cpp.md).

## State

`Stateless.` The callback's name is a field of the effector.

## `CAnimatorCamEffectorScriptCB::Valid`

**Contract** — reports whether the effector should stay on the stack, and **fires the
script callback exactly once** on the transition to invalid. The callback name is cleared
as it fires, which is the latch: a cyclic or repeatedly-queried effector cannot call the
script twice.

**Invariants** — the named script function must exist; a missing one is a hard failure
rather than a silent skip, because a cutscene whose end handler never runs leaves the
player without control and the failure would be diagnosed far from its cause.

```text
FUNCTION is_valid() -> bool
  valid = base validity        # false once the animation has played out
  IF NOT valid AND a callback name is still set THEN
    call the named global script function
    clear the callback name    # fire once, never again
  RETURN valid
```

## `CAnimatorCamEffectorScriptCB::ProcessIfInvalid`

**Contract** — keeps positioning the camera after the animation has ended, but only in
absolute-positioning mode: the camera is pinned to the animation's final transform, with
the field-of-view override still applied if one was set. In relative mode the effector does
nothing once invalid and the camera returns to the player.

**Notes** — this is what makes a cutscene hold its last frame instead of snapping back
while the script decides what to do next. The pairing is the design: the callback tells the
script the move is over, and the hold gives the script a frame or more to act before the
view changes.
