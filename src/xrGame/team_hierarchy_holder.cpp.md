# src/xrGame/team_hierarchy_holder.cpp

> One team's squads, created the first time anybody asks for one.

**Needs** — [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md)
**Used by** — [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md)
**Tier floor** — T2: a fixed-capacity sparse registry.

## Purpose

The engine organises creatures into a three-level hierarchy — **team**, then **squad**, then
**group** — and every creature carries its coordinates in it. This is the middle level: one
team's squad registry. The level above holds teams, the level below holds groups, and each
level is the same shape, so reading this one explains all three.

The hierarchy is what group behaviour is addressed through: the combat assessment that drives
panic asks *my group*, the cover bookkeeping asks *my squad*. A creature's coordinates are
authored in its spawn record and are stable for its life.

## State

```text
RECORD TeamHierarchyHolder
  team   : reference to the team registry above       # the parent; never none
  squads : list<optional<SquadHierarchyHolder>>       # exactly 256 slots, mostly empty
```

**Invariants** — the registry is a **fixed 256 slots, pre-sized and null-filled at
construction**. A squad identifier is therefore an index, not a key: lookup is an array read
with no search and no hashing, which matters because the lookup is on the path of every
group query in combat. The cap of 256 is the width of the squad identifier in the spawn
record and is frozen by the shipped data.

Most slots stay empty for the whole game. The registry trades 256 empty references per team
— a handful of teams exist — for a constant-time lookup, which is the right trade at this
size.

## `squad(id)`

**Contract** — the squad registry for an identifier, creating it on first request. Fails if
the identifier is out of range. Never returns nothing: asking about a squad is what brings it
into existence.

```text
FUNCTION squad(id) -> SquadHierarchyHolder
  REQUIRE id < 256
  IF squads[id] IS none
    squads[id] := new SquadHierarchyHolder(parent: self)
  RETURN squads[id]
```

**Invariants** — creation on read is the decision here, and it is what removes the entire
registration problem. Nothing ever has to declare that a squad exists: a creature spawned
with squad coordinates simply asks for its squad and the chain — team, squad, group — is
built down from the top on demand. Nothing is ever removed either; a squad whose last member
dies keeps its registry, ready if a member is spawned into it later.

The out-of-range check reports the offending identifier in its message. Squad coordinates
come from authored spawn data, so an out-of-range one is a data error, and the number is what
a designer needs to find it.

## Teardown

**Contract** — destroying the team registry destroys every squad registry it created. There
is no way to release one squad.
