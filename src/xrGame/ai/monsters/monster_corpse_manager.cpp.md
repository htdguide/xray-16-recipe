# src/xrGame/ai/monsters/monster_corpse_manager.cpp

> Holds the one corpse a creature is currently interested in: normally the nearest one its memory offers, but overridable by script, in which case memory is ignored until the body is picked clean.

**Needs** — [`monster_corpse_manager.h`](monster_corpse_manager.h.md) · [`monster_corpse_memory.h`](monster_corpse_memory.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`monster_corpse_manager.h`](monster_corpse_manager.h.md)
**Tier floor** — T3: a selection and a cached snapshot

## Purpose

[`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md) knows about many corpses; the
creature's eating state wants exactly one, and wants it to stay the same between ticks unless
something changed. This is the one-element layer between them, and it is also where script
takes control.

The pattern — a *memory* that accumulates and a *manager* that selects one and can be
overridden — repeats for enemies ([`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md)
over [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md)). Reading either explains the
other.

## State

```text
RECORD CorpseManager
  creature        : BaseMonster
  corpse          : optional<entity>
  position        : vector          # snapshot at selection time
  vertex          : int
  time_last_seen  : int
  forced          : bool            # script chose this corpse; memory is not consulted
```

Invariant: while `forced` is set, `position`, `vertex` and `time_last_seen` are frozen at the
moment of forcing and are never refreshed. While it is clear, all three are copied from the
memory's answer every tick.

## `update`

**Contract** — re-selects the current corpse. Called once per creature update. Allocates
nothing.

```text
FUNCTION update()
  IF forced
    IF corpse.food_remaining < 1        # picked clean: drop it, but stay in forced mode
      corpse = none
    RETURN

  corpse = memory.best_corpse()
  IF corpse
    snapshot = memory.best_corpse_info()
    position, vertex, time_last_seen = snapshot
```

**Notes** — in forced mode the only escape is the body running out of food. Losing sight of it,
its retention expiring, or another creature claiming it do not clear the selection; a scripted
creature walks to its assigned meal regardless. Note also that clearing the selection leaves
`forced` set, so the creature then has no corpse at all until script intervenes — it does not
fall back to memory. That asymmetry is the point of the mode.

## `force_corpse` / `release_corpse`

**Contract** — `force_corpse` pins a specific body, snapshotting its live position, navigation
vertex and the current clock, and enters forced mode. `release_corpse` leaves forced mode and
immediately re-selects from memory, so the creature does not spend a tick with nothing chosen.

**Notes** — `force_corpse` snapshots the corpse's *live* position, whereas the memory path
snapshots the position at last sighting. A forced corpse is therefore always located exactly,
even one the creature has never seen.

## `reinit` / `forget_entity`

**Contract** — `reinit` clears the selection and leaves forced mode; used when the creature is
re-created from its record. `forget_entity` clears the selection if it names the entity being
destroyed, but leaves `forced` set — so a scripted creature whose target was destroyed stays
in forced mode with nothing selected, exactly as when the body is picked clean.
