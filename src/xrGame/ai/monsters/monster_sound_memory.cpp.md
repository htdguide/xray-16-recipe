# src/xrGame/ai/monsters/monster_sound_memory.cpp

> Turns the engine's sound events into a creature's short-term hearing: a filtered, deduplicated, continuously rescored list of what it has heard, plus a separate one-shot signal that a pack-mate is in trouble.

**Needs** — [`monster_sound_memory.h`](monster_sound_memory.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`xrServerEntities/ai_sounds.h`](../../../xrServerEntities/ai_sounds.h.md) · [Seam: Audio device](../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`monster_sound_memory.h`](monster_sound_memory.h.md)
**Tier floor** — T3: a short list, a linear scan for the maximum, and a per-tick rescore

## Purpose

Perception of sound in this engine is *event-driven*: an emitter carries AI-perception
attributes and the engine delivers an event to every listener in range. This file is a
creature's end of that. It does four things:

1. **Classifies** the engine's bit-flag sound type into the ordered danger rank declared in
   [`monster_sound_memory.h`](monster_sound_memory.h.md).
2. **Filters** — most world sound is thrown away on arrival.
3. **Deduplicates by (source, rank)**, so a burst of gunfire from one shooter is one memory
   that keeps being refreshed, not a hundred.
4. **Rescores** everything every tick, because the score depends on distance and age and the
   creature moves.

A fifth, genuinely separate thing also lives here: the **help call**, a one-shot latch that
remembers the navigation vertex at which a pack-mate was heard fighting. It shares the file but
not the list.

## State

```text
RECORD SoundMemory
  creature        : BaseMonster
  retention       : int                  # milliseconds a sound survives
  sounds          : list<HeardSound>     # reserved for 20; no hard cap
  help_call_time  : int                  # 0 means no live help call
  help_call_vertex: int
```

Invariants: no two entries share both a source and a rank. Every entry's rank is strictly
before `door_opening` and is not `not_dangerous`. A help call is live only within a fixed
window — ten seconds, a constant of this file — of `help_call_time`, and the latch is
self-clearing.

## `classify`

**Contract** — reduce the engine's sound type, a set of bit flags, to one danger rank. Pure.

```text
FUNCTION classify(engine_type) -> DangerRank
  IF engine_type carries none of { weapon, monster, world }
    RETURN not_dangerous

  test the specific flags IN THIS ORDER, returning on the first that is present:
    weapon_recharging, weapon_shooting, item_taking, item_hiding,
    weapon_empty_clicking, weapon_bullet_hit,
    monster_dying, monster_injuring, monster_step,
    monster_talking, monster_attacking,
    world_object_breaking, world_object_colliding

  RETURN not_dangerous
```

**Invariants** — the *order of the tests* is the classification rule, because the engine's type
is a set of flags and a sound may carry several. A shot that is also a monster attack classes
as a shot. Reordering the tests changes which rank a compound sound gets.

**Notes** — three ranks in the enumeration — `weapon_changing`, `monster_jumping`,
`monster_falling` — have no test here and so are unreachable. They still occupy positions, and
because the dangerous/discarded cut points are expressed against *neighbouring* members, those
positions matter even though nothing ever lands on them. Removing them in a rebuild shifts the
cuts.

`item_taking` and `item_hiding` map to the *weapon* taking and hiding ranks, so a creature
hears any item being picked up as a weapon being drawn.

## `hear`

**Contract** — two forms. The five-argument form classifies, scores against the creature's
current position, and delegates. The one-argument form filters and admits.

```text
FUNCTION hear(sound)
  IF sound.rank = not_dangerous                THEN RETURN   # nothing to learn
  IF sound.rank is at or after door_opening    THEN RETURN   # world noise: discarded
  IF sound.rank = monster_walking AND no source THEN RETURN  # anonymous footsteps

  replaced = false
  FOR EACH entry IN sounds
    IF entry.source = sound.source AND entry.rank = sound.rank
      IF sound.time >= entry.time
        entry = sound
        replaced = true

  IF NOT replaced THEN append sound
```

**Invariants** — deduplication is on the pair (source, rank), so one entity can occupy several
entries with different ranks at once — shooting and walking simultaneously is two memories.

**Notes** — the discard of everything from `door_opening` onward is what makes creatures deaf
to the world: doors, breaking glass and falling objects reach this routine and die here. That
is a deliberate scope reduction, not a gap.

The anonymous-footstep rejection exists because a footstep with no attributable source gives a
creature a position to investigate and nobody to blame, which the brains handle badly.

The dedup loop does not stop at the first match and the replacement flag is set inside it, so a
list that somehow held two entries for one pair would have both replaced. Harmless; the
invariant prevents it.

## `update_hearing`

**Contract** — the per-tick step: evict, rescore, and expire the help-call latch. Called once
per creature update.

```text
FUNCTION update_hearing()
  remove every entry whose source has been destroyed,
                      or whose source is an entity that is no longer alive,
                      or that is older than the retention window

  FOR EACH entry IN sounds
    recompute entry.score against the creature's CURRENT position and the current clock

  IF help_call_time + HELP_CALL_WINDOW < now THEN help_call_time = 0
```

**Notes** — rescoring every entry every tick is what makes the ranking track the creature as it
moves; without it a sound scored when the creature was far away would stay low even after it
closed. It is the reason the score is a stored field rather than computed at query time — the
queries want a plain maximum.

Removing sounds made by entities that have died is how a creature stops investigating a fight
that is over.

## Queries

- **`most_dangerous_sound`** — the entry with the greatest score. The name is inherited; as
  established in [`monster_sound_memory.h`](monster_sound_memory.h.md), the rank does not enter
  the score, so this is really "the loudest, nearest, freshest".
- **`most_dangerous`** — the same entry plus the boolean *dangerous*, which is the genuine rank
  test: at or before `weapon_empty_clicking`. So the pair is "the top-scoring sound, and
  whether its rank is threatening", and those two facts are decided by different rules.
- **`first_sound`** — the oldest surviving entry, with the same dangerous flag. Used where a
  creature should react to what it heard *first* rather than to what is most urgent.
- **`sound_from(entity)`** — the first entry made by a given entity, whatever its rank. This is
  what lets the enemy manager fuse a sound's position into its target's last known location.
- **`is_loud_sound(threshold)`** — whether any entry's power exceeds a value. A pure magnitude
  test, ignoring rank, distance and age.

All of these require a non-empty list; the callers check first.

## The help call

**Contract** — a separate, single-slot latch. `note_help_call(engine_type, vertex)` records the
navigation vertex if the sound qualifies and no call is already live; `heard_help_call` answers
whether one is live; `help_call_vertex` yields it.

```text
FUNCTION note_help_call(engine_type, vertex)
  IF engine_type lacks monster_attacking THEN RETURN
  IF engine_type lacks monster_injuring  THEN RETURN
  IF engine_type lacks monster_dying     THEN RETURN

  IF help_call_time is non-zero THEN RETURN     # first call wins until it expires

  help_call_time   = now
  help_call_vertex = vertex
```

**Notes** — the three tests are conjunctive: a sound must carry **all three** flags — attacking
*and* injuring *and* dying — to register as a call for help. No shipped sound carries all
three, so the latch never fires and the whole help-call mechanism is dead in the shipped game.
The intent is plainly the disjunction ("a pack-mate attacking, hurt, or dying"). It is recorded
here as it is, because a rebuild that "fixes" it switches on a behaviour the shipped levels
were never balanced against; the honest move is to reproduce the dead path and offer the
disjunction as an option.

The latch is first-wins rather than newest-wins: while a call is live, further calls are
ignored, so a creature runs to the first cry it heard and not to wherever the fight moved.

The window is ten seconds, a constant of this file, and it is both how long a call stays live
and what `update_hearing` uses to clear it. A rebuild should keep the two identical.

## `forget_entity`

**Contract** — removes every entry made by the departing entity. Does not touch the help-call
latch, which stores a vertex rather than a reference and so has nothing to clear.
