# src/xrGame/seniority_hierarchy_holder.cpp

> The root of the command hierarchy: hands out team holders by index, creating each on first demand and owning them all.

**Needs** — [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: an owning registry with a defined destruction point

## Purpose

Creatures in this engine are grouped four deep — team, then squad within the team, then
group within the squad, then the members — and that grouping is what the combat AI reasons
about when it decides who is an ally, who to report an enemy to, and who shares a danger.
This class is the top of that tree and the only part of it the level owns directly: the
level creates one at load and destroys it at unload, and everything below hangs off it.

The three levels below are the same shape with different capacities and live in their own
files; splitting them is arbitrary and a rebuild may express all four as one parameterized
registry.

## State

```text
RECORD SeniorityHierarchyHolder
  teams : list<optional<TeamHolder>>   # exactly 64 slots, most of them absent
```

**Invariants** — the list is always exactly at capacity (see
[`seniority_hierarchy_holder_inline.h`](seniority_hierarchy_holder_inline.h.md)); a team
index is a slot index, not a position. A slot, once filled, is never emptied while the
holder lives — teams are created and never retired, because a team that briefly has no
members must keep its history for the members who rejoin.

## `team`

**Contract** — returns the team holder for an index, creating it if this is the first ask.
The new holder is given a back-reference to this one, so a creature holding only its group
can walk back up to the level's root. Asking for an index at or beyond capacity is a
programming error and fails loudly with the offending index in the message; it is not
clamped, because a spawn record with an out-of-range team is data corruption that must be
found, not absorbed.

```text
FUNCTION team(index) -> TeamHolder
  REQUIRE index < 64             # FAIL WITH "team id is invalid: <index>"
  IF teams[index] IS absent
    teams[index] = new TeamHolder(parent: self)
  RETURN teams[index]
```

**Notes** — creation on demand rather than up front is what keeps the cost proportional to
the teams a level actually uses; sixty-four fully built four-level trees would be a large
allocation for a level with two factions.

## Destruction

**Contract** — deletes every present team holder, which recursively deletes the squads,
groups and member lists below. This is the single point at which the whole hierarchy goes
away, and it must run *after* every creature has been unregistered — a creature that
outlived its group holder would hold a reference into freed memory. The level's teardown
order guarantees this.
