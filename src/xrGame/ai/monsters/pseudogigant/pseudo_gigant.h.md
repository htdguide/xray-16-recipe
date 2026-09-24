# src/xrGame/ai/monsters/pseudogigant/pseudo_gigant.h

> Declares the pseudogiant: a creature whose distinctiveness is entirely in the ground — its footfalls shake the camera and its stomp is an area attack that throws physics objects and staggers the player.

**Needs** — [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md)
**Used by** — [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) · [`pseudogigant_script.cpp`](pseudogigant_script.cpp.md) · [`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md)
**Tier floor** — T3: a creature type with two extra abilities and their tuned parameters

## Purpose

Declares the surface implemented in [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md). The giant is
the chapter's clearest example of "a creature is the shared base plus one distinctive ability
and a section of numbers": its brain in
[`pseudogigant_state_manager.cpp`](pseudogigant_state_manager.cpp.md) is the baseline selector
with no giant-specific state at all, and everything that makes a giant a giant is on this page.

It also mixes in the controlled-entity facet, which is what lets a controller take it over as a
puppet — the giant is one of the creatures the controller can turn against the player.

## State

```text
RECORD PseudoGigant
  # authored, from the creature's configuration section
  step_shake            : { time, amplitude, periods }   # the camera shake per footfall
  stomp_effector        : screen-and-camera effect description, from a named second section
  stomp_damage          : real                # at point blank; falls off with distance
  stomp_particles       : text                # effect asset name
  stomp_range           : { min, max }        # the stomp is only offered inside this band
  stomp_cooldown        : { min, max }        # a random delay in this band after each stomp
  stagger_duration      : int (milliseconds)  # how long the player's acceleration is locked
  jump_velocities       : two velocity profiles, prepare and landing
  # runtime
  next_stomp_allowed_at : int
  nearby                : scratch list of objects in stomp range
```

**Invariants** — `stomp_cooldown` is sampled once per stomp into `next_stomp_allowed_at`, so
the interval between stomps is random within the authored band rather than fixed. The minimum
range is as load-bearing as the maximum: a giant standing on top of the player cannot stomp,
which is what forces it to back off and re-approach.

## Exported units

- `Load` — the animation table, the shake parameters, the stomp effect and every stomp number.
- `reinit` — the jump velocity profiles and the rotation-jump and stomp animation bindings.
- `ability_earthquake` — declares to the shared base that this creature's footfalls shake
  things; the base is what calls the footfall hook.
- `event_on_step` — a footfall landed.
- `check_start_conditions` — the veto on motion-control requests: this is where the stomp's
  range band and cooldown are enforced.
- `on_activate_control` — a motion-control request was granted; start the wind-up sound and
  arm the cooldown.
- `on_threaten_execute` — the stomp lands.
- `HitEntityInJump` — damage applied when the leap connects, read from the leap animation's
  own authored parameters.
- `TranslateActionToPathParams` — the giant never runs; see the implementation.
- `get_monster_class_name` — the name the script and configuration layers know it by.
