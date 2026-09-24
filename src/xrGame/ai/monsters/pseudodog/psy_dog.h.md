# src/xrGame/ai/monsters/pseudodog/psy_dog.h

> Declares the psi dog and its phantom: a pseudodog that conjures illusory copies of itself and hides behind them, and the copy that materialises in the player's face and vanishes when struck.

**Needs** — [`psy_dog.cpp`](psy_dog.cpp.md) · [`pseudodog.h`](pseudodog.h.md) · [`psy_dog_aura.h`](psy_dog_aura.h.md)
**Used by** — [`pseudodog_script.cpp`](pseudodog_script.cpp.md) · [`psy_dog.cpp`](psy_dog.cpp.md) · [`psy_dog_aura.cpp`](psy_dog_aura.cpp.md) · [`psy_dog_state_manager.cpp`](psy_dog_state_manager.cpp.md)
**Tier floor** — T2: spawns and destroys entities at runtime and owns a screen effect on the player

## Purpose

Declares the two types implemented in [`psy_dog.cpp`](psy_dog.cpp.md). This is the chapter's
best example of a creature that is *one distinctive ability plus a base*: everything about the
psi dog's body, animation, movement and fighting is the pseudodog's, and what it adds is the
phantom mechanic.

## `PsyDog`

The real creature. It keeps a population of phantoms alive while it has an enemy, refuses to
expose itself while too few of them are up, and projects a screen effect on the player whenever
the player and the phantoms have recently noticed each other.

Its own surface:

- `Load`, `reinit`, `reload` — the aura's effect section and the phantom parameters.
- `think` — the per-tick step: drive the aura, respawn phantoms, or wipe them all.
- `phantom_count`, `must_hide` — the population and the question the brain branches on.
- `create_state_manager` — substitutes the psi dog's brain over the pseudodog's body.
- `on_death`, `on_despawn` — both destroy every phantom.
- private `spawn_phantom`, `register_phantom`, `unregister_phantom`, `destroy_all_phantoms`.

It holds: the aura, the live phantom list, the authored minimum and maximum populations, the
respawn delay, and a fixed array of per-slot death timestamps that is the respawn schedule.

## `PsyDogPhantom`

A conjured copy, and itself a pseudodog — it walks, leaps and fights with the same body. What
distinguishes it:

- it **spawns invisible and disabled**, and materialises only once it is facing the enemy it
  inherited from its parent, with a leap, a particle burst and a camera and screen jolt on the
  player;
- it **takes its target from its parent**, not from its own perception;
- **any hit from its target destroys it** — it has no health in practice;
- it destroys itself if it strays more than a fixed distance from its parent.

Its surface is the spawn path, the per-tick step, the hit reaction, the two destruction paths
(self-initiated and parent-initiated), and the parent-registration handshake.
