# src/xrGame/ai/monsters/controller/controller_psy_hit.h

> Declares the controller's set-piece psi attack, implemented in [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md).

**Needs** — [`../control_combase.h`](../control_combase.h.md)
**Used by** — [`controller.cpp`](controller.cpp.md) · [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md) · [`controller_tube_inline.h`](controller_tube_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControllerPsyHit`, a custom control element the controller registers on its first
creature-specific channel. Substance is in
[`controller_psy_hit.cpp`](controller_psy_hit.cpp.md).

## State

Four clips, the current stage index, a five-state sound machine, the authored minimum
distance, a flag recording whether the player's weapons are currently blocked, and the
cooldown stamp. The element has **no channel payload**: it has no per-use parameters, because
its target is always the actor.

Exported units:

- `load`, `reinit`, `update_frame` (empty), `check_start_conditions`, `activate`,
  `deactivate`, `on_event` — the ability lifecycle, driven entirely by animation-end events.
- `on_death` — abort cleanly when the creature is killed mid-attack, so the player is not
  left disarmed with a camera effector running.
- `tube_ready` — the cooldown check, which the creature also exposes.
- `stop`, `play_anim`, `death_glide_start`, `death_glide_end`, `set_sound_state`, `hit`,
  `check_conditions_final`, `see_enemy` — private; the stage transitions and the effects on
  the player.

The separation of `stop` from `deactivate` is load-bearing: the first undoes what was done to
the player's view, the second what was done to the player's input and to the creature's body.
Both are reachable independently and both are idempotent.
