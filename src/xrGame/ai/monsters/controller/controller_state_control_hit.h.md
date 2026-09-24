# src/xrGame/ai/monsters/controller/controller_state_control_hit.h

> Declares the controller's mind-control strike state, implemented in
> [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md)
**Used by** — [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the surface of the state that plays the controller's signature attack: a wind-up, a strike
at a fixed moment inside the wind-up animation, and a wait for the animation to finish. Split
from its body only because C++ splits templates that way.

## `CStateControlAttack`

- **enter** — arm the internal phase machine
- **execute** — advance one phase per tick, and each tick re-assert idle posture, facing and sound
- **leave** (clean and forced) — nothing beyond the base contract
- **is_finished** — the phase machine reached its end
- **may_start** — far enough away and currently seeing the enemy

Its private state is the phase marker and the moment the wind-up began. Contracts are in the
implementation twin.
