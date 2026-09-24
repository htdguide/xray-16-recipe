# src/xrGame/ai/monsters/states/monster_state_eat.h

> Declares the feeding behaviour: find a corpse, approach it, check it, eat, and refuse to be hungry again for a while.

**Needs** — [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`bloodsucker_state_manager.cpp`](../bloodsucker/bloodsucker_state_manager.cpp.md) · [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md) · [`burer_state_manager.cpp`](../burer/burer_state_manager.cpp.md) · [`cat_state_manager.cpp`](../cat/cat_state_manager.cpp.md) · [`chimera_state_manager.cpp`](../chimera/chimera_state_manager.cpp.md) · [`controller_state_manager.cpp`](../controller/controller_state_manager.cpp.md) · [`flesh_state_manager.cpp`](../flesh/flesh_state_manager.cpp.md) · [`fracture_state_manager.cpp`](../fracture/fracture_state_manager.cpp.md) · [`poltergeist_state_manager.cpp`](../poltergeist/poltergeist_state_manager.cpp.md) · [`pseudodog_state_manager.cpp`](../pseudodog/pseudodog_state_manager.cpp.md) · [`pseudogigant_state_manager.cpp`](../pseudogigant/pseudogigant_state_manager.cpp.md) · [`snork_state_manager.cpp`](../snork/snork_state_manager.cpp.md) · [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) · [`tushkano_state_manager.cpp`](../tushkano/tushkano_state_manager.cpp.md) · _and 1 more_
**Tier floor** — T3: a container over a corpse reference with a satiety timer

## Purpose

Declares the surface implemented in
[`monster_state_eat_inline.h`](monster_state_eat_inline.h.md). Registered by nearly every
creature in the chapter and selected by their brains as the lowest-priority thing to do when
nothing is happening, which is what makes an area feel inhabited rather than idle.

## State

```text
RECORD EatState
  corpse            : optional<entity>   # the chosen body; a real entity reference
  last_eaten_at     : int                # when feeding last finished
```

The satiety window after a meal is a compiled-in twenty seconds.

**Invariants** — `corpse` is a cached reference to another entity, which is why this state is
one of the few in the chapter whose `remove_links` does real work rather than merely cascading:
a corpse can be destroyed while being eaten, and the reference must be dropped.

The satiety timer is what stops a creature from re-entering feeding the instant it leaves —
without it, the brain's lowest-priority branch would select eating again on the next tick and
the creature would never do anything else while a body lay nearby.

## Exported units

- construction and destruction.
- `reinit` — clear the corpse reference and the satiety timer across a respawn or save load.
- `initialize` / `finalize` / `critical_finalize` — claim and release the chosen corpse.
- `remove_links` — drop the corpse reference if it names the entity being destroyed.
- `reselect_state` / `setup_substates` — the approach, inspect, eat, and walk-away phases.
- `check_start_conditions` — is there a corpse worth eating, and am I hungry.
- `check_completion` — the meal is over.
- `hungry` (private) — the satiety test.
