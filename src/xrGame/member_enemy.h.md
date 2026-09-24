# src/xrGame/member_enemy.h

> One enemy as a squad sees it: who knows about him, who has been assigned to him, and how likely he is to be the one worth shooting.

**Needs** — [`member_enemy_inline.h`](member_enemy_inline.h.md) · [`memory_space.h`](memory_space.h.md) · [`entity_alive.h`](entity_alive.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`member_enemy_inline.h`](member_enemy_inline.h.md)
**Tier floor** — T3: a record with an ordering

## Purpose

A squad fights as a unit, which means it must share what its members know and then divide the
work. This record is one enemy's entry in that shared picture: which members can see him,
which members have been given him as a target, where he was last seen, when, and the squad's
confidence that he is the right thing to attack.

The mask fields are the whole idea. A squad is a fixed set of members, each with a bit, so
"who knows" and "who is assigned" are single integers rather than sets — set union, assignment
and membership are all one instruction, and the squad coordinator performs those operations
per enemy per member per frame.

## State

```text
RECORD MemberEnemy
  object          : EntityAlive      # the enemy
  mask            : bits             # which squad members are aware of him
  distribute_mask : bits             # which squad members have been assigned to him
  probability     : real = 1.0       # the squad's confidence that this is the right target
  enemy_position  : (real, real, real)   # last agreed position
  level_time      : int = 0          # when that position was agreed
```

**Invariant** — a newly constructed entry is known by exactly the member who reported it,
assigned to nobody, and at full confidence. Confidence is lowered later, as the report ages or
is contradicted.

**Invariant** — the awareness mask and the assignment mask are independent. A member may know
about an enemy without being assigned to him — which is the normal case for everyone but the
one or two members actually engaging — and the coordinator relies on the distinction to spread
fire rather than have the whole squad shoot one target.

**Invariant** — the squad member mask width caps the squad size. It is the same width used
throughout the memory system, which is where the type comes from, and it is the reason a squad
cannot grow past that many members.

## `CMemberEnemy` construction, equality and ordering

**Contract** — construct from an enemy and the mask of the member who reported him. Equality
is against the bare enemy, so the coordinator finds an existing entry by naming the enemy and
merges the new reporter's bit into it rather than creating a second entry.

**Ordering is by descending confidence** — an entry with higher probability sorts *first*.
That inversion is deliberate and is the file's single most consequential line: the coordinator
sorts the shared enemy list and then walks it front to back handing out assignments, so the
most-likely-correct target is distributed first and gets the most shooters. A rebuild that
sorts ascending inverts the squad's entire targeting priority.
