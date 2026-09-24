# src/xrGame/PHMovementDynamicActivate.cpp

> Changing a character's collision box without pushing it through the world, and remembering when that failed.

**Needs** — [`PHMovementControl.h`](PHMovementControl.h.md) · [`xrPhysics/PHCharacter.h`](../xrPhysics/PHCharacter.h.md) · [`xrPhysics/MovementBoxDynamicActivate.h`](../xrPhysics/MovementBoxDynamicActivate.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: iterative penetration resolution against a live solver

## Purpose

A character's posture — standing, crouching, prone — is a choice among authored collision
boxes, and switching to a *larger* one can leave the body overlapping geometry. Standing up
under a ledge must **fail**, not eject the character through the floor.

This file is the careful switch. It is separate from the rest of the movement layer because
its concern is exactly one thing: attempt a resize, and if it cannot be resolved, put
everything back exactly as it was.

The iterative resolution itself belongs to the physics layer; what lives here is the
transaction around it — the save, the restore, and the memory of failure that stops a
creature retrying an impossible posture every frame.

## State

Owned by the movement layer, used only here:

```text
trying_times : 4 timestamps   # when each box last failed to activate
trying_poses : 4 positions    # and where the body was when it did
```

**Invariants** — a failure is remembered per box *with a position*, so it expires either
after half a second or as soon as the body has moved five centimetres. Both conditions must
hold to suppress a retry: a stationary creature under a ledge does not retry, a creature that
stepped out from under it does.

## `ActivateBoxDynamic`

**Contract** — attempt to make a numbered box the active one, resolving any penetration the
change creates. Returns whether it succeeded. On failure nothing is changed — box, position,
velocity and body existence are all as they were.

```text
FUNCTION activate_box_dynamic(id, iterations = 9, steps = 5, resolve_depth = 0.01)
  # 1. is this attempt already known to fail?
  IF a body exists AND this box has a remembered failure
      AND it was under 500 ms ago
      AND the body has moved less than 0.05 since
    RETURN false

  # 2. this only applies to character bodies, not to full physical shells
  IF no character body OR the object has a full physics shell
    RETURN false
  IF the box is already active AND a body exists
    RETURN true

  # 3. a body may not exist yet — create a temporary one to test against
  body_was_there = a body existed
  body_was_disabled = it existed and was asleep
  IF NOT body_was_there
    create the body

  save velocity and position

  succeeded = resolve the resize, letting the solver separate the body from
              geometry over `iterations` iterations of `steps` sub-steps,
              accepting a residual overlap of `resolve_depth`

  IF NOT succeeded
    IF the body was ours to create -> destroy it
    ELSE IF it was asleep          -> put it back to sleep
    restore the previous box
    restore the velocity
    straighten the body's rotation        # the resolve can leave it tipped
    restore the position
  ELSE
    commit the new box

  restore the velocity                    # again, on both paths
  IF failed AND the body was already there
    remember this failure: time and position
  ELSE
    clear this box's failure record
  RETURN succeeded
```

**Invariants**
- The undo must restore *body existence* as well as state: a probe that created a body to
  test with must destroy it, and one that woke a sleeping body must put it back. Otherwise a
  failed attempt to stand leaves the character awake and simulating for no reason.
- The rotation is straightened explicitly on failure. The penetration resolve applies impulses
  to separate the body, and a character capsule that has been tipped does not right itself —
  it is a permanent, visible bug.
- A failure is only remembered when the body already existed. A probe that had to create its
  own body is not evidence about a state the creature was actually in.

**Notes** — the four tuning arguments — nine iterations, five sub-steps, a centimetre of
accepted residual overlap — are the resolve's budget. They are defaulted at the declaration
and the default is what every caller uses, so they are effectively constants; nothing states
how they were chosen.

The velocity is restored twice on the failure path, once inside the undo and once after. It is
idempotent, not a decision.

The refusal when the object has a full physics shell rather than a character body is the
boundary of this mechanism: a ragdolled or fully simulated object has no posture to switch.
