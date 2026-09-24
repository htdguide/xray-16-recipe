# src/xrGame/ai/monsters/snork/snork_jump.cpp

> Entirely commented out. The only thing recoverable from it is the flanking pounce the snork was going to have and does not.

**Needs** — [`snork_jump.h`](snork_jump.h.md)
**Used by** — [`snork_jump.h`](snork_jump.h.md)
**Tier floor** — T3: dead code; nothing to implement

## Purpose

Every line of this file is inside a comment. It compiles to nothing, is referenced by nothing,
and describes a leap controller that was superseded by the shared motion-control machinery.
A rebuild deletes it. The page exists so the mirror is complete and so the abandoned design is
recorded once rather than rediscovered.

## The design that was abandoned

The intent is legible and worth one paragraph, because it is a behaviour the snork visibly
lacks. The controller would have chosen between two leaps by the angle to the target: if the
enemy was within thirty degrees of the snork's facing, an ordinary pounce straight at it;
otherwise a *flanking* leap — aimed not at the enemy but at a point four units in front of the
enemy along the enemy's own facing and two units up, so the snork would land in front of a
player who was turning away rather than behind them. The flanking leap would then be re-aimed
mid-air: while airborne it would keep probing forward, abort if the target receded past ten
units, and re-launch if it closed inside two.

Two of the three animation clips it names are the ones the shipped leap uses; the third, a
sideways-jump clip, is named only by the abandoned flanking path and suggests the manoeuvre had
an animation authored for it.

## Notes

Nothing here should be revived on the strength of this file alone. The shipped snork leaps
using the shared machinery, and re-introducing a second leap would need the mid-air re-aiming
to interact with the path builder, which is precisely the problem the shared machinery exists
to solve.
