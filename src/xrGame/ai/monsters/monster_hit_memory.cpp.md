# src/xrGame/ai/monsters/monster_hit_memory.cpp

> Who has hurt this creature lately, and from which of four sides, so a creature that is shot from behind can turn the right way without ever having seen the shooter.

**Needs** — [`monster_hit_memory.h`](monster_hit_memory.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md)
**Used by** — [`monster_hit_memory.h`](monster_hit_memory.h.md)
**Tier floor** — T3: a short list and a linear scan for the newest entry

## Purpose

A hit is the one form of perception that cannot be missed, and it is what drives a creature's
reaction to an attacker it has not detected. This is the record of them.

Two decisions carry the file. First, the memory is **keyed by attacker**, one entry each: a
burst of ten rounds from one source leaves one entry, refreshed, not ten. What a brain wants
from this is "who and when", not "how often".

Second, what is recorded is not a direction but a **side** — front, back, left, right — and the
direction is reconstructed later by rotating the creature's *current* facing by a quarter, a
half or three quarters of a turn. So a creature that has turned since being hit derives a
direction relative to where it is facing now, not where the hit actually came from. That is
almost certainly a simplification for cheapness, and it is visible in play as creatures that
spin in place when hit repeatedly while turning.

Note also that the *position* recorded is the creature's own position at the moment of the hit,
not the attacker's. "Where I was standing when it happened" is what the fleeing behaviours want.

## State

```text
RECORD Hit
  source   : optional<entity>   # who did it; may be nothing for environmental damage
  position : vector             # where the CREATURE was, not where the attacker was
  time     : int                # global clock
  side     : enum { front, back, left, right }

RECORD HitMemory
  creature  : BaseMonster
  retention : int               # milliseconds; default 10000, overridden at bind
  hits      : list<Hit>         # at most one entry per distinct source
```

Invariant: no two entries share a source. The eviction pass also removes any entry whose source
has since died, so a creature stops reacting to a dead attacker without waiting out the
retention.

## `record_hit`

**Contract** — admit one hit from a source, on a side. Refreshes the existing entry for that
source if there is one, otherwise appends. Stamps the creature's current position and the
global clock. Allocates only on a first hit from a new source.

## `update` / eviction

**Contract** — removes every entry that has aged past the retention window **or** whose source
is an entity that is no longer alive. Called once per creature update.

**Notes** — the age comparison is written with the retention and the timestamp on the same side
of the addition, which for a global clock that always exceeds the retention behaves as
intended. It is fragile only against a clock that starts near zero.

## The "last hit" queries

**Contract** — four separate readers, each scanning the list for the newest entry and returning
one field of it: the source, the time, the creature's position at that hit, and the derived
direction. Each scans independently; there is no cached newest.

**`last_hit_direction`** is the only one with an algorithm:

```text
FUNCTION last_hit_direction() -> direction
  direction = creature.facing                  # the default when nothing is remembered
  newest    = the entry with the greatest time, or none

  IF newest exists
    heading, pitch = decompose(direction)
    CASE newest.side OF
      back  : heading = heading + HALF_TURN
      left  : heading = heading + QUARTER_TURN
      right : heading = heading - QUARTER_TURN
      front : unchanged
    RETURN normalised direction from (heading, pitch)

  RETURN direction
```

**Notes** — the pitch is carried through unchanged from the creature's current facing, so a
hit from above or below is not representable at all; only the four horizontal quadrants are.

A memory with no hits answers the creature's own facing, which is a usable default rather than
an error — callers do not have to check `has_hits` first, though the flee behaviours do anyway.

The newest entry is found by a full scan, repeated once per query. With at most a handful of
distinct attackers this is cheaper than maintaining a cache; a rebuild may keep an index if the
list is allowed to grow.

## `was_hit_by` / `forget_entity`

**Contract** — `was_hit_by` answers whether a given entity has an entry. `forget_entity`
removes it. Both compare by identity.
