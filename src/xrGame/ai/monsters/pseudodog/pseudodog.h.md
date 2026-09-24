# src/xrGame/ai/monsters/pseudodog/pseudodog.h

> Declares the pseudodog: a leaping pack predator that can drag corpses and perform a psi howl, and the base the psi dog extends.

**Needs** — [`pseudodog.cpp`](pseudodog.cpp.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md)
**Used by** — [`pseudodog.cpp`](pseudodog.cpp.md) · [`pseudodog_script.cpp`](pseudodog_script.cpp.md) · [`pseudodog_state_manager.cpp`](pseudodog_state_manager.cpp.md) · [`psy_dog.cpp`](psy_dog.cpp.md) · [`psy_dog.h`](psy_dog.h.md)
**Tier floor** — T2: a creature type registering animation and jump data at load

## Purpose

Declares the surface implemented in [`pseudodog.cpp`](pseudodog.cpp.md). The pseudodog is
close to the chapter's baseline creature: it has the full posture set (standing, sitting,
lying, sleeping), it fights in melee, and almost everything about it is the shared creature
base driven by its own configuration section.

Three things are its own, and they are what the class adds:

- **it can drag a corpse**, which gives it the dragging gait and the drag animation;
- **it can psi-attack** — a distinct howl animation and sound, used as an intimidation display
  and, in the derived psi dog, as the cover for something else;
- **it leaps**, through the shared jump controller, with a rotating variant that lets it change
  facing mid-leap by a quarter turn.

It is also the base of [`psy_dog.h`](psy_dog.h.md), which is where the interesting behaviour
lives.

## Exported units

- `Load`, `reload`, `reinit` — the creature's own animation set, sounds, jump data and the two
  anger thresholds.
- `can_drag`, `can_psi_attack` — capability answers, both yes, which the shared states query.
- `handle_special_animation_flags(flags)` — the hook the animation layer calls when a clip's
  authored flags request the psi attack or the threat display.
- `hit_entity_in_leap(entity)` — the damage applied when a leap connects.
- `create_state_manager` — builds the brain, so a derived class can substitute its own.
- Public fields `anger_hunger_threshold`, `anger_loud_threshold`, `became_angry_at`,
  `growling_since` — authored numbers and timers for an anger model that is not implemented.
  See [`pseudodog.cpp`](pseudodog.cpp.md).
- A sound tag `psi_attack`, declared as a species extension above the custom base.
