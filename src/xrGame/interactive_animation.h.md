# src/xrGame/interactive_animation.h

> Declares an animated physics body that stops its own animation when it hits something.

**Needs** — [`physics_shell_animated.h`](physics_shell_animated.h.md) · [`interactive_animation.cpp`](interactive_animation.cpp.md)
**Used by** — [`CharacterPhysicsSupport.cpp`](CharacterPhysicsSupport.cpp.md) · [`interactive_animation.cpp`](interactive_animation.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in
[`interactive_animation.cpp`](interactive_animation.cpp.md). A lighter mechanism than
[`interactive_motion.h`](interactive_motion.h.md): where that one hands a body to ragdoll,
this one merely fades the animation out and stops driving.

Exported units:

- `interactive_animation` — an animated body bound to one specific blend.
- `update` — advance one frame; answers whether the animation is still running.
- `create_shell` — build the body and install the contact filter.
- `collide` — whether the body is interpenetrating anything by more than the tolerance.
- `contact_callback` — the contact filter; records the deepest penetration of the frame.
