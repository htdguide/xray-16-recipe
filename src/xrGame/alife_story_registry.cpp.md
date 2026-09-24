# src/xrGame/alife_story_registry.cpp

> The index that lets a script address an entity by the name a level designer gave it, rather than by a runtime identifier nobody can predict.

**Needs** — [`alife_story_registry.h`](alife_story_registry.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — reached through its declarations in [`alife_story_registry.h`](alife_story_registry.h.md); callers name that, not this file.
**Tier floor** — T2: a filtered map insertion

## Purpose

A **story identifier** is an authored constant attached to a spawn record: the designer
marks one particular stalker, one particular door, one particular stash as story-relevant,
and the scripts then refer to it by that constant. Runtime entity identifiers are
allocated and cannot be written into a script; story identifiers are authored, declared in
configuration and exported to the script layer as an enumeration (see
[`alife_simulator_script.cpp`](alife_simulator_script.cpp.md)).

This registry is the map between the two. It is offered every entity as it registers, and
keeps the ones that carry a story identifier.

## `add`

**Contract** — indexes an entity under its story identifier. An entity with no story
identifier is ignored. A collision is a fault unless the caller tolerates it.

```text
FUNCTION add(story_id, object, tolerate_collision)
  IF story_id is the invalid value -> RETURN
  IF story_id already indexed
    IF NOT tolerate_collision -> FAIL WITH "story object already in the registry"
    RETURN                      # tolerated: the first entity keeps the identifier
  objects.insert(story_id, object)
```

**Invariants** — the identifier is **unique across the whole world**, not per level. Two
entities sharing one is normally a data error — the designer marked two things with the
same constant, and the scripts addressing it would get whichever registered first.

The tolerant form exists for the one legitimate case: a level transition can briefly have
both the outgoing and the incoming instance of a story entity present. In that window the
first keeps the identifier and the second is silently not indexed, which is correct only
because the first is about to leave.

The invalid-identifier guard is checked first, and this is why the registry can be offered
every entity in the world without the callers filtering: the vast majority carry no story
identifier and fall straight through.

## Notes

The diagnostic build logs every insertion with the entity's display name and the level its
game-graph vertex belongs to. That is deliberately verbose — story entities are few and
their placement is exactly what a designer debugging a quest needs to see — and it is one
of the few log lines in the alife layer worth keeping in a rebuild.
