# src/xrGame/sight_manager_inline.h

> The cheap queries on the aiming manager, the argument-forwarding order constructors, and the switch that arms the bone solver.

**Needs** — [`sight_manager.h`](sight_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors and one assignment

## Purpose

Carries the bodies for [`sight_manager.h`](sight_manager.h.md). Two things here are
load-bearing: the default when no order is chosen, and what arming the bone solver
actually sets.

## `use_torso_look`

**Contract** — asks the chosen order whether the torso should follow the head. With no
order chosen the answer is **true**.

**Invariants** — the default is not arbitrary. This flag selects between two sets of
head/shoulder/spine blend factors in [`sight_manager.cpp`](sight_manager.cpp.md); the
torso-look set is the one that twists the body rather than craning the neck. A creature
with no look order should stand square, not with its head turned, so the safe default is
the one that keeps the head aligned with the body.

## `setup` (forwarding forms)

**Contract** — three forms taking one, two or three arguments and forwarding them
unchanged to a look order's constructors, then issuing that order. They exist so callers
across the AI layer can write a look order inline without naming the order type; which
sight type results is decided entirely by the constructor overloads described in
[`sight_action_inline.h`](sight_action_inline.h.md).

## `bone_aiming`

**Contract** — two forms. The no-argument form **disarms** the solver: it clears the clip
name and sets the aiming type to none, which routes the per-frame computation down the
factor-blended path instead. The three-argument form arms it with a clip name, which end
of that clip to match, and which solver (weapon or head). Arming with "none" as the solver
is a programming error — disarming has its own form.

**Invariants** — the armed form does not validate that the named clip exists or is
playing; the solver does that later and fails loudly. What matters here is that the clip
name and the frame selector are set *together*, because a solver armed with one and not
the other would match against the wrong pose.

## Rotation accessors

**Contract** — `current_spine_rotation`, `current_shoulder_rotation` and
`current_head_rotation` return the smoothed additive rotations. They are read by the
animation layer every frame while composing the creature's pose, so they must be plain
reads with no computation.

## `turning_in_place` / `enabled`

**Contract** — field reads. `turning_in_place` is consulted by the movement and animation
layers to pick a turning clip; `enabled` gates the whole aiming update.
