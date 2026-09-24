# src/xrGame/ai/monsters/controller/controller_state_attack_moveout.h

> Declares the "steal toward where the enemy was" state, implemented in
> [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md).

**Needs** — [`../state.h`](../state.h.md) · [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md)
**Used by** — [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md) · [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Names the surface of the state the controller uses to leave cover and creep toward a lost enemy.
Declaration and body are split only because C++ splits templates that way.

## `CStateControlMoveOut`

- **enter** — prime the path builder, latch the enemy's current navigation vertex, start in the
  first of two approach phases
- **execute** — advance the phase, drive the path and the stealth posture, and re-pick a look
  direction on a timer
- **is_finished** / **may_start** — the sight-based predicates

Its private state is the two-phase marker, the current path target (position plus vertex), the
latched enemy vertex, and the look-point with its own refresh clock. Contracts are in the
implementation twin.
