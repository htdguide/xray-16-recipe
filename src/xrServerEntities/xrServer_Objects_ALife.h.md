# src/xrServerEntities/xrServer_Objects_ALife.h

> Declares the alife record hierarchy — what makes a record something the off-screen simulation owns — plus the two things this header itself decides: the editor's lookup tables and the group template.

**Needs** — [`xrServer_Objects.h`](xrServer_Objects.h.md) · [`alife_space.h`](alife_space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`restriction_space.h`](restriction_space.h.md)
**Used by** — [`BreakableObject.cpp`](../xrGame/BreakableObject.cpp.md) · [`Car.h`](../xrGame/Car.h.md) · [`CarWheels.cpp`](../xrGame/CarWheels.cpp.md) · [`ClimableObject.cpp`](../xrGame/ClimableObject.cpp.md) · [`GameObject.cpp`](../xrGame/GameObject.cpp.md) · [`Grenade.cpp`](../xrGame/Grenade.cpp.md) · [`HangingLamp.cpp`](../xrGame/HangingLamp.cpp.md) · [`Helicopter.cpp`](../xrGame/Helicopter.cpp.md) · [`InventoryBox.cpp`](../xrGame/InventoryBox.cpp.md) · [`Missile.cpp`](../xrGame/Missile.cpp.md) · [`PHDestroyable.cpp`](../xrGame/PHDestroyable.cpp.md) · [`PHSkeleton.cpp`](../xrGame/PHSkeleton.cpp.md) · [`PhysicObject.cpp`](../xrGame/PhysicObject.cpp.md) · [`PhysicObject.h`](../xrGame/PhysicObject.h.md) · _and 22 more_
**Tier floor** — T2: a hierarchy declaration; the byte layouts are in the implementation.

## Purpose

Declares the spine of the chapter: the ladder from "a record in a spawn file" up to "a
record the alife simulation schedules, moves across the game graph and switches online and
offline". Contracts and serialized layouts are in
[`xrServer_Objects_ALife.cpp`](xrServer_Objects_ALife.cpp.md); two units are defined here and
so are documented here.

## The ladder — what each level adds

Read downward; each row keeps everything above it.

| Level | Adds |
|---|---|
| base record | identity, placement, class, custom-data overlay, client blob |
| **alife object** | a game-graph vertex, a level vertex, the alife flag word, story identifiers, a per-record random generator |
| **dynamic object** | a timestamp and a switch counter — the right to be brought online and taken offline, and to attach and detach children |
| **dynamic visual object** | a model |
| **physics-skeleton object** | a saved ragdoll pose |
| **space restrictor** | a volume and a restrictor kind |
| **smart zone** | schedulability: the simulation calls it, it hands out jobs |

Orthogonal to the ladder are three *mixins* that a record adds by multiple inheritance rather
than by descent: schedulable (the simulation updates me), group abstract (I am a population
that breeds), and the volume and visual mixins from the chapter's base files. The hierarchy
is wide and non-virtual on purpose; see
[`smart_cast.h`](smart_cast.h.md) for the cost that decision imposes and how it is paid.

## `CSE_ALifeGroupTemplate`

**Contract** — defined here because it is a template: it welds any record type to the
group-population mixin, and its every method is "call the record's, then call the group's".
The **order is the contract**: the record's payload comes first in the stream, the group's
second, in all four directions. It also re-routes construction — a group record's
configuration section may name a *different* section for the individual (`monster_section`),
so that "a pack of dogs" and "a dog" are separate configuration entries.

**Notes** — a rebuild with composition writes this as a wrapper that owns both parts and
serializes them in order. The template exists only because C++ has no other way to add a
mixin to an arbitrary base and keep the non-virtual layout.

## `SFillPropData`

**Contract** — the editor's shared lookup tables, reference-counted: every place name per
location axis, every level's caption, every story identifier, every spawn-story identifier,
every character profile, and every smart-cover description name. Loaded on the first record
that needs them and released when the last such record is destroyed.

**Invariants**

- The four location-type axes are read from numbered sections of the game configuration; a
  missing section is fatal, because a graph point with no vocabulary cannot be edited.
- The story and spawn-story lists are sorted by name and get a synthetic first entry meaning
  "none", whose value is all-bits-set — the same value the record stores for "no story".
- Smart-cover names come from a **Lua table**, not from configuration: the editor asks the
  script engine for `smart_covers.descriptions` and enumerates its keys. The tools therefore
  need a live script virtual machine. They are sorted with a *natural* ordering (so that
  `loophole_2` precedes `loophole_10`), which is available only on one platform; elsewhere
  they are left unsorted. Cosmetic.

**Notes** — reference counting here is not about memory; it is about *when the script engine
and configuration are available*. The tables cost a noticeable load and are wanted only while
records that use them exist. A rebuild may simply load them once with the editor.

## Exported units

Every entry below is a concrete record type whose fields and serialization are in
[`xrServer_Objects_ALife.cpp`](xrServer_Objects_ALife.cpp.md).

- `CSE_ALifeSchedulable` — the "the simulation updates me" mixin, with the evaluation-function
  type queries a creature uses to rank weapons, detectors, anomalies and other creatures.
- `CSE_ALifeGraphPoint` — a vertex of the cross-level game graph, authored into the level.
- `CSE_ALifeObject` — the alife base.
- `CSE_ALifeGroupAbstract` — a breeding population.
- `CSE_ALifeDynamicObject` — online/offline capable.
- `CSE_ALifeDynamicObjectVisual` — the above plus a model.
- `CSE_ALifePHSkeletonObject` — the above plus a ragdoll pose.
- `CSE_ALifeSpaceRestrictor` — a volume that constrains movement.
- `CSE_ALifeLevelChanger` — a restrictor that moves the actor to another level.
- `CSE_ALifeSmartZone` — a schedulable restrictor: the base of smart terrain.
- `CSE_ALifeObjectPhysic` — a freely simulated prop.
- `CSE_ALifeObjectHangingLamp` — a light with a breakable model.
- `CSE_ALifeObjectProjector` — a searchlight.
- `CSE_ALifeHelicopter`, `CSE_ALifeCar` — vehicles.
- `CSE_ALifeObjectBreakable`, `CSE_ALifeObjectClimable` — destructible and ladder.
- `CSE_ALifeMountedWeapon`, `CSE_ALifeStationaryMgun` — emplaced weapons.
- `CSE_ALifeTeamBaseZone` — a multiplayer base volume.
- `CSE_ALifeInventoryBox` — a container.
