# src/xrGame/ai/monsters/monster_sound_memory.h

> Declares the heard-sounds memory: the classification of a sound into a danger rank, the score that ranks one sound against another, and the separate "a pack-mate is in trouble" signal.

**Needs** — [`monster_sound_memory.cpp`](monster_sound_memory.cpp.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`xrServerEntities/ai_sounds.h`](../../../xrServerEntities/ai_sounds.h.md)
**Used by** — [`base_monster.h`](basemonster/base_monster.h.md) · [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md) · [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md) · [`monster_sound_memory.cpp`](monster_sound_memory.cpp.md)
**Tier floor** — T3: a small list, a linear scan, and an integer score

## Purpose

Declares the surface implemented in
[`monster_sound_memory.cpp`](monster_sound_memory.cpp.md), and — because they are data rather
than behaviour — carries the danger ranking and the scoring formula itself.

## `DangerRank`

The ordered classification every heard sound is reduced to. **The order is the ranking**: the
enumeration is listed most dangerous first, and both the danger question and the score read it
positionally, so reordering it changes creature behaviour.

```text
ENUM DangerRank              # most dangerous first — the ORDER is load-bearing
  weapon_shooting
  monster_attacking
  weapon_bullet_ricochet
  weapon_recharging
  weapon_taking
  weapon_hiding
  weapon_changing
  weapon_empty_clicking      # --- everything at or before here counts as DANGEROUS ---
  monster_dying
  monster_injuring
  monster_walking
  monster_jumping
  monster_falling
  monster_talking
  door_opening               # --- everything at or after here is DISCARDED on arrival ---
  door_closing
  object_breaking
  object_falling
  not_dangerous              # the catch-all
```

**Invariants** — two cut points, both expressed as comparisons against a member rather than as
a stored flag:

- **Dangerous** means at or before `weapon_empty_clicking`. So every weapon sound including an
  empty click is dangerous, and a dying creature is not. A creature reacts to the *threat* of a
  weapon, not to the evidence of harm.
- **Discarded** means at or after `door_opening`. Doors and falling objects are heard by the
  engine and thrown away here, so creatures never investigate world noise. Only the tail
  `not_dangerous` sits past them, and it is discarded by its own explicit test.

## `HeardSound`

```text
RECORD HeardSound
  source   : optional<entity>   # who made it; the sound's own position is separate
  rank     : DangerRank
  position : vector             # where the SOUND was, not where its maker now is
  power    : real               # the emitter's loudness
  time     : int                # global clock when it was heard
  score    : int                # the ranking; recomputed every tick
```

## The score

**Contract** — an integer combining four terms with fixed weights. Higher is more urgent; the
memory's "most dangerous sound" is the maximum.

```text
score = 8  * (index of not_dangerous - index of weapon_shooting)   # a constant: the rank span
      - 1  * floor(distance from the listener to the sound)
      - 2  * floor(whole seconds since it was heard)
      + 50 * floor(power)
```

**Notes** — the first term is written as though it scaled with the sound's own rank, but both
indices are *fixed members of the enumeration*, so it evaluates to the same constant for every
sound and contributes nothing to the ordering. The rank therefore does not enter the score at
all: ranking among remembered sounds is decided purely by distance, age and loudness, and the
rank only decides the binary dangerous/not question elsewhere. This is near-certainly a
mistake — the intent reads as "danger rank times eight" — and it is what the shipped creatures
behave by. Reproduce it and flag it.

Of the three terms that do matter, power dominates by a wide margin: one unit of loudness is
worth fifty world units of distance or twenty-five seconds of age. Both age and power are
floored to integers before weighting, so a sound under one second old and one just under a
second old score identically, and a power under one contributes nothing.

## Exported units

- `bind`, `hear(sound)` / `hear(source, engine_type, position, power, time)`, `has_sounds`,
  `count`.
- `first_sound`, `most_dangerous_sound`, `most_dangerous`, `sound_from` — the queries.
- `update_hearing` — the per-tick evict and rescore.
- `is_loud_sound(threshold)` — whether anything remembered exceeds a power.
- `clear`, `forget_entity`.
- `heard_help_call`, `help_call_vertex`, `note_help_call` — the separate pack-distress signal.
