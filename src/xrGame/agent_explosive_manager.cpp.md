# src/xrGame/agent_explosive_manager.cpp

> Registers a live grenade as a danger area for the whole squad, and decides which single member shouts the warning.

**Needs** — [`agent_explosive_manager.h`](agent_explosive_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`danger_explosive.h`](danger_explosive.h.md) · [`danger_object_location.h`](danger_object_location.h.md) · [`member_order.h`](member_order.h.md) · [`Missile.h`](Missile.h.md) · [`Explosive.h`](Explosive.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a small assignment search plus one danger-area registration per grenade

## Purpose

Two jobs, and they are usefully separate. When a squad member notices a live explosive:

1. **The place becomes dangerous for everyone.** A danger area is registered with the
   squad's location manager, so the pathfinder's cost model steers every member away from
   it for as long as the grenade can still go off. This happens once, immediately, and does
   not depend on who reacts.
2. **Exactly one member reacts visibly** — the shout, the dive. That is an assignment
   problem identical to the one in
   [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md), and it is solved the same
   way.

## State

```text
RECORD ExplosiveEntry
  explosive   : explosive       # the explosive behaviour
  game_object : object          # the thing in the world carrying it
  reactor     : optional<creature>   # the member assigned to react
  time        : int             # global clock reading at registration

RECORD ExplosiveManager
  explosives     : list<ExplosiveEntry>   # pending, unassigned sightings
  already_seen   : list<entity id>        # every explosive ever registered by this squad
  squad          : agent
```

Invariants:

- The pending list is a work queue: an entry with a reactor is handed off and removed.
- The *already seen* list is not a queue — it is a suppression set. An explosive is
  registered by a squad **once, ever**, and the identifier stays in the set until the
  object leaves the simulation. Without it, every member who notices the same grenade
  would register a fresh danger area and a fresh reaction.
- An entry is keyed by the explosive for the duplicate check and by the world object's
  entity identifier for the suppression set. The two are different handles on the same
  thing, and both checks run.

## `register_explosive`

**Contract** — a squad member has noticed a live explosive. Ignored if the explosive is
already pending, or if this squad has ever registered it. Otherwise the object is added to
the suppression set and the pending list, and a danger area is registered with the squad's
location manager: centred on the object, starting now, lasting for a computed interval,
with a fixed radius.

**Invariants** — the danger's lifetime is the heart of the function:

```text
FUNCTION register_explosive(explosive, object)
  IF explosive is already pending THEN RETURN
  IF object.id IN already_seen THEN RETURN
  already_seen.add(object.id)
  pending.add(explosive, object, reactor: none, time: now)

  lifetime = GRENADE_AFTERMATH                       # 1 second
  IF the explosive is a thrown missile with a known detonation time in the future THEN
    lifetime = (detonation_time - now) + GRENADE_AFTERMATH
  squad.location.add_danger_area(object, from: now, for: lifetime, radius: GRENADE_RADIUS)
```

- A grenade with a known fuse keeps the area dangerous until it detonates *plus* one
  second; anything else (a mine, a barrel, an explosive whose timer is already past) gets
  the one second alone. The extra second is the aftermath: creatures should not walk into
  the spot the instant the blast ends, because the blast has a duration of its own and the
  reaction animations need somewhere to finish.
- The danger radius is ten metres, a fixed constant with no derivation beyond being
  roughly a grenade's lethal radius. It is *not* read from the grenade's own blast radius,
  so every explosive produces the same avoidance area regardless of size. A rebuild should
  take the radius from the explosive.

## `process_explosive` / `react_on_explosives`

**Contract** — the same nearest-visible assignment as the corpse manager, with the same
displacement rule and the same iterate-to-a-fixed-point loop: each pending explosive is
matched to the nearest combat member who can see it and is not already reacting to
something, then each assignment is handed to that member as a pending grenade reaction
carrying the explosive, its world object and the sighting time, and the assigned entries
are removed.

**Invariants** — identical to the corpse manager's, including the detail that only
*combat* members are considered and that a member already reacting is skipped.

**Notes** — this is the second copy of the same algorithm. The two differ only in which
list they scan, which reaction slot they fill and which fields they copy across. A rebuild
should write it once, parameterized by the pending record and the reaction slot; that also
fixes both copies of the loop-termination mistake described in
[`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md).

## `remove_links`

**Contract** — drops an object from both the pending list and the suppression set when it
leaves the simulation.

**Invariants** — removing from the suppression set is what makes a grenade's identifier
reusable. Entity identifiers are recycled, so leaving a dead grenade's identifier in the
set would make the squad ignore a future explosive that happened to reuse it.

## `update`

**Contract** — does nothing; the manager is event-driven. See the note in
[`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md).
