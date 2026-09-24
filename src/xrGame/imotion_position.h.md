# src/xrGame/imotion_position.h

> Declares the death-animation variant that drives the body by moving it to the animated pose, with a look-ahead that rejects poses which would intersect the world.

**Needs** — [`interactive_motion.h`](interactive_motion.h.md) · [`imotion_position.cpp`](imotion_position.cpp.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`imotion_position.cpp`](imotion_position.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`imotion_position.cpp`](imotion_position.cpp.md), which
holds the substance.

## State

```text
RECORD PositionMotion
  update_callback : TracksUpdateHook   # installed into the animation system; every
                                       # advance of the animation tracks is routed
                                       # through this motion instead
  time_to_end     : real               # animation time remaining, in seconds of real time
  saved_visual_callback : optional<Callback>   # the owner's normal post-pose hook, removed
                                               # for the duration and restored at the end
  blend           : optional<Blend>    # the death animation's blend, found by searching
  shell_has_history : bool             # whether the body has accumulated motion history
```

**Invariants** — the animation's own update hook is *replaced*, not wrapped. For the duration
of the death animation the animation system does not advance the tracks on its own; this
motion advances them, in sub-steps of its choosing, and tests each one. That inversion of
control is the design, and it is why the class needs a hook object rather than a method.

## Exported units

Every member is private; the base class drives all of them.

- `state_start` / `state_end` — take over and hand back the skeleton, the bone-update
  pipeline, the body's enabled state and the contact filter.
- `move_update` — the per-frame hook: recompute the pose once with updates re-enabled.
- `move` — advance the animation by one frame, in bounded sub-steps.
- `motion_collide` — one sub-step: advance, collide, and decide whether to continue.
- `collide_animation` — advance the animation and run a collision test at the resulting pose.
- `advance_animation` — advance the tracks and evaluate the pose, without colliding.
- `collide_not_move` — test the current pose without committing to any advance.
- `force_calculate_bones` — evaluate the whole skeleton with the suppression lifted.
- `disable_update` — arm or disarm the interception.
- `init_bones` / `deinit_bones` / the root-bone callback — apply the authored heading offset.
- `collide` — deliberately empty; collision is inside the move.
