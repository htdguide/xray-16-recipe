# src/xrGame/ai/monsters/controlled_actor.h

> Declares the actor-hold mixin, implemented in [`controlled_actor.cpp`](controlled_actor.cpp.md).

**Needs** — [`actor_input_handler.h`](../../actor_input_handler.h.md)
**Used by** — [`bloodsucker.h`](bloodsucker/bloodsucker.h.md) · [`bloodsucker_alien.cpp`](bloodsucker/bloodsucker_alien.cpp.md) · [`controlled_actor.cpp`](controlled_actor.cpp.md) · [`controller.cpp`](controller/controller.cpp.md) · [`controller.h`](controller/controller.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlledActor`, an actor input handler that a creature class inherits in order
to take the player's camera and input. Substance is in
[`controlled_actor.cpp`](controlled_actor.cpp.md).

## State

The camera target point, a per-axis arrival flag, and a flag asking that the turn continue.
A run-lock trio is declared and never used.

Exported units:

- `install` (with and without an actor), `release`, `reinit` — take and give back the hold.
- `frame_update` — ease the camera one frame.
- `look_point` — name the point to turn toward; called every tick while holding.
- `dont_need_turn` — freeze the camera where it is, for when a camera effector is taking
  over the view.
- `authorized` — the input whitelist: selecting the first weapon slot, and firing while it
  is active. Everything else is refused.
- `mouse_scale_factor` — unbounded, which kills the player's aim.
- `is_turning`, `is_installed`, `is_controlling` — queries.

The creature that inherits this is [`controller/controller.h`](controller/controller.h.md);
inheritance rather than composition is how "the actor is held by *this* creature" is
answered without a registry.
