# src/xrGame/ai/monsters/controller/controller_state_attack_hide_lite.h

> Declares the "hide only until the enemy loses sight of me" variant, implemented in
> [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md)
**Used by** — [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md) · [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the surface of a second, cheaper run-to-cover state whose success condition is *breaking
line of sight* rather than *reaching the chosen cell*. The declaration is separate from the body
only because C++ splits templates that way.

## `CStateControlHideLite`

- **reset** — clear the finish stamp
- **enter** — choose a cover point and prime the path builder
- **execute** — drive movement, animation, sound and head-look each tick
- **leave** — stamp the finish time
- **is_finished** — arrived *or* no longer seen
- **may_start** — unconditionally yes

Its private state is the chosen target (position plus navigation vertex) and the finish stamp.
Contracts are in the implementation twin.
