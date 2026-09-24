# src/xrGame/ik_limb_state.h

> The saved state of one limb between frames, and the rule that decides when it has gone stale.

**Needs** — [`ik_calculate_data.h`](ik_calculate_data.h.md) · [`ik_calculate_state.h`](ik_calculate_state.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`IKLimb.cpp`](ik/IKLimb.cpp.md) · [`IKLimb.h`](ik/IKLimb.h.md) · [`ik_limb_state.cpp`](ik_limb_state.cpp.md)
**Tier floor** — T2: a state holder plus a staleness predicate and a seeding rule

## Purpose

Foot placement is continuous across frames: this frame's solve starts from last frame's
answer. This file is the container that carries the answer across, and — more importantly —
it holds the two rules that govern the hand-off: **when is last frame's answer still usable**,
and **what does a new solve inherit from it**. Both are here rather than in
[`ik_limb_state.cpp`](ik_limb_state.cpp.md), which holds only the frame-conversion arithmetic.

## State

```text
RECORD LimbState
  state : CalculateState      # the full per-limb state; see ik_calculate_state.h
  limb  : optional<Limb>      # which limb, needed to convert between reference bones
```

## `state_valide`

**Contract** — the saved state is usable when it was computed recently enough:

```text
FUNCTION is_valid(state) -> bool
  RETURN now <= state.calc_time + update_interval + last_frame_duration
```

**Invariants** — this is the staleness rule and it is the reason a character who was offscreen,
paused, or simply not updated for a few frames does not snap a foot to a placement computed
long ago. The allowance is the solver's own update interval **plus one frame's duration**: the
interval is how often the limb is *supposed* to be solved, and the frame duration covers the
case where the last frame ran long. A rebuild that uses a fixed timeout will either discard
valid state on a slow frame or accept stale state on a fast one.

An invalid state is not cleared; it is simply not inherited from. The distinction matters
because the state is still readable for debugging.

## `get_calculate_state`

**Contract** — seeds a new frame's working state from the saved one. This is the selective
inheritance and the selection is the substance:

```text
FUNCTION seed(new_state)
  new_state.calc_time = now

  # a new solve is "blending" if the old one was, OR if the footfall flag just changed
  new_state.blending = is_valid(saved)
                       AND (saved.blending OR saved.foot_step != new_state.foot_step)

  # the collision placement carries over, CONVERTED to the new reference bone
  new_state.collide_pos = saved.collide_pos converted to the current reference bone

  new_state.speed_blend_l = saved.speed_blend_l    # blend rates persist
  new_state.speed_blend_a = saved.speed_blend_a
  new_state.unstuck_time  = saved.unstuck_time     # so does the release timestamp
```

**Invariants**:

- **A change in the footfall flag starts a blend.** The frame a foot is declared planted, or
  released, is exactly the frame the placement must begin moving smoothly rather than
  jumping. That single condition is what makes every footfall transition continuous.
- **Nothing is inherited at all when the saved state is stale**, because the blending flag is
  gated on validity. A stale limb starts a fresh, unblended solve — a snap — which is correct:
  blending toward a placement computed seconds ago would be worse.
- The goals themselves are **not** inherited here. Only the collision placement, the blend
  rates and the release time carry over; the goal is recomputed. That is what keeps the solve
  authoritative rather than drifting.

## Reference-bone conversion

**Contract** — every placement the state holds is expressed against a *reference bone*, which
may change between frames. The accessors — the animation placement, the goal, the blend
target and the ground-search direction — each convert on the way out, using the saved
bone-to-bone transform. The implementation is in
[`ik_limb_state.cpp`](ik_limb_state.cpp.md).

**Invariants** — conversion happens on **read**, not on write. The state stores whatever the
solve that produced it used, and every consumer asks for it in the frame it needs. A rebuild
that normalizes on write must then also store which bone it normalized to, and gains nothing.

## `save_new_state`

**Contract** — replaces the saved state wholesale. There is no merge: the new solve has
already inherited what it needed through the seeding above.

## Exported queries

- `ref_bone` — which bone the stored placements are against.
- `foot_step`, `blending` — the two flags a caller needs without pulling the whole state.
- `valide` — the staleness test above, as a method.
