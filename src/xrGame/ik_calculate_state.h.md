# src/xrGame/ik_calculate_state.h

> The per-limb state the foot-placement solver carries between frames: where the foot was put, what it collided with, and what it is blending toward.

**Needs** — _(none)_
**Used by** — [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`ik_calculate_data.h`](ik_calculate_data.h.md) · [`ik_limb_state.h`](ik_limb_state.h.md)
**Tier floor** — T2: a state record; no device contact

## Purpose

Foot placement is not a per-frame function of the pose. A placed foot must stay placed while
the animation moves around it, must blend smoothly toward a new placement rather than
snapping, and must remember how it was constrained so the next frame can decide whether that
constraint still holds. This file is that memory. It has no implementation file: the record
*is* the contract.

## State

```text
ENUM CollideState
  free           # nothing under the foot: the animation's placement stands
  rotational     # the ground constrains the foot's ORIENTATION only
  translational  # the ground constrains the foot's POSITION only
  mixed          # both, from different contacts
  aligned        # the foot is flat on a single surface
  undefined      # not yet computed this frame

RECORD GoalMatrix
  transform     : matrix        # a placement for the foot, in world space
  collide_state : CollideState  # HOW that placement was arrived at

RECORD CalculateState
  calc_time     : int               # level time the current goal was computed
  unstuck_time  : int               # level time of the last forced release;
                                    # all-ones means "never"
  goal          : GoalMatrix        # where the foot is being driven to now
  blend_to      : GoalMatrix        # where it will be driven next, blended toward
  anim_pos      : matrix            # the placement the raw animation asked for, kept so the
                                    # solver can fall back to it and so the IK correction
                                    # can be measured as a delta
  collide_pos   : GoalMatrix        # the placement collision alone produced
  b2tob3        : matrix            # the fixed transform between the two end bones of the
                                    # limb; see below
  pick          : vector = (0,-1,0) # the direction the ground is searched along — straight
                                    # down by default, but a limb on a slope or a wall
                                    # overrides it
  speed_blend_l : real              # how fast the position is allowed to converge
  speed_blend_a : real              # ... and the orientation
  foot_step     : bool              # the animation says this foot is planted
  idle          : bool
  blending      : bool
  ref_bone      : int (16-bit)      # which bone of the limb the goal is expressed against;
                                    # none = all-ones
```

**Invariants** — the load-bearing ones:

- **A placement is never just a transform.** Every goal carries how it was constrained,
  because the next frame's decision depends on it: a foot placed by a purely rotational
  constraint may be slid freely, one placed translationally may not. A rebuild that stores
  bare transforms will have to recompute the classification, and the classification depends
  on contacts that are gone by then.
- **Two goals, not one.** The current goal and the goal being blended toward are separate, so
  that a new ground contact retargets the blend without discarding the placement the foot is
  currently honouring. Collapsing them makes every new contact a snap.
- **Two blend speeds, not one.** Position and orientation converge at different rates because
  a foot that rotates as fast as it translates reads as a foot sliding on ice.
- **The reference bone is part of the state.** A limb's goal can be expressed against either
  of its last two bones — the ankle or the toe — and which one is correct depends on how the
  foot is contacting the ground. The stored transform between those two bones is what lets the
  goal be converted from one to the other without re-deriving it from the pose.
- **The unstuck timestamp is all-ones, not zero, when unset.** Zero is a valid level time, and
  "was released at time zero" and "was never released" must be distinguishable.

The collision state starts *undefined* rather than *free*: not knowing whether the ground is
there is different from knowing it is not, and the solver treats them differently.

## `ik_goal_matrix`

**Contract** — a placement and its classification, settable only as a pair. That is the whole
point of the type: it is impossible to store a placement without saying how it was
constrained.

The default is the identity transform with an undefined classification, which the solver
reads as "no placement has been computed".
