# src/xrPhysics/MovementBoxDynamicActivate.h

> The port through which physics reaches back into a character's movement control
> to resize its collision box, plus the procedure that does the resizing safely.

**Needs** — [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md) · [`PHCharacter.h`](PHCharacter.h.md)
**Used by** — [`PHMovementControl.h`](../xrGame/PHMovementControl.h.md) · [`PHMovementDynamicActivate.cpp`](../xrGame/PHMovementDynamicActivate.cpp.md) · [`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md)
**Tier floor** — T2: an interface of five calls and one procedure declaration.

## Purpose

A character's collision volume is not fixed. Standing, crouching and crawling are different
boxes, and switching between them cannot be done by assignment: the new box may intersect
geometry the old one did not, and a character that grows into a ceiling must be pushed down
rather than shoved through it. This header declares both halves of that transaction — the
small interface physics needs on the game's movement controller, and the procedure that
performs the switch.

This is the [`xrEngine` ⇄ `xrPhysics` cycle](../../SYSTEM-REQUIREMENTS.md#7-build-order) in
miniature, and it is worth reading as the pattern: physics does not know what a movement
controller is, only that something can hand it a character, a list of candidate boxes, and a
way to interpolate between two of them.

## `IPHMovementControl`

**Contract** — what a movement controller must offer for its box to be resized. Five
obligations:

- **`character`** — the controller's underlying character body. Physics drives this
  directly during the switch.
- **`actor_calculate`** — run one tick of the controller's own movement logic with a given
  desired acceleration, camera direction, angular speed, jump request and timestep. The
  resize procedure calls this with zeroed inputs to keep the controller's internal state
  consistent while the world is being stepped underneath it.
- **`Boxes`** — the authored set of candidate collision boxes, indexed by an identifier the
  caller supplies (stand, crouch, crawl, and so on).
- **`Box`** — the box currently in effect.
- **`InterpolateBox`** — set the live box to a blend between the current one and candidate
  *id*, at a fraction from 0 to 1. This is how the resize is done gradually rather than at
  once.

**Invariants** — `InterpolateBox` must be monotone in its fraction and must reach candidate
*id* exactly at 1. The resize procedure calls it with fractions `1/n, 2/n, … 1` and relies
on the last call leaving the box exactly equal to the target; a rebuild that eases the
interpolation will leave the character in a box nobody authored.

**Notes** — `actor_calculate` taking a camera direction is not an accident of the signature:
a character's movement logic consults the look direction for climbing and jumping, so a tick
run without one would take a different branch. The resize procedure passes a fixed unit
direction rather than the real one, which is safe only because it also passes zero
acceleration and no jump.

## `ActivateBoxDynamic`

**Contract** — switch a character to candidate box *id*, pushing it out of anything the new
box intersects, and report whether it converged. Declared here, implemented in
[`MovementBoxDynamicActivate.cpp`](MovementBoxDynamicActivate.cpp.md). The iteration count,
the number of growth steps and the acceptable residual penetration are parameters, and the
caller also states whether the character's body already exists — which changes all three,
because a character being *created* has no previous box to grow from.
