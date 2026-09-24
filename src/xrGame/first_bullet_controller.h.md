# src/xrGame/first_bullet_controller.h

> Declares the multiplayer-only "first shot is accurate" rule implemented in [`first_bullet_controller.cpp`](first_bullet_controller.cpp.md).

**Needs** — [`first_bullet_controller.cpp`](first_bullet_controller.cpp.md)
**Used by** — [`Weapon.cpp`](Weapon.cpp.md) · [`Weapon.h`](Weapon.h.md) · [`first_bullet_controller.cpp`](first_bullet_controller.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the small value object a weapon embeds to grant a reduced-dispersion first shot to
a player who has held fire and is moving slowly. Substance is in
[`first_bullet_controller.cpp`](first_bullet_controller.cpp.md).

The load-bearing fact the declaration carries is its *placement*: this is a plain member of
the weapon, not a separate registered object, so its timer is per weapon instance rather
than per player. Two weapons carried by one player each track their own idle time.

Exported units:

- `first_bullet_controller` — holds the timestamp of the last shot, the idle timeout the
  privilege requires, the dispersion to substitute, the actor speed limit above which the
  privilege lapses, and the enable flag read from the weapon's section.
- `load` — reads the four tuning values from a weapon section, and reads nothing further
  when the feature is disabled there.
- `is_bullet_first` — the predicate, taking the shooter's current linear speed.
- `get_fire_dispertion` — the substitute dispersion to use when the predicate holds.
- `make_shot` — stamps the current time, ending the privilege until the timeout elapses
  again.
