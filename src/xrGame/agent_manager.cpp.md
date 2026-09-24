# src/xrGame/agent_manager.cpp

> The squad brain: one shared decision-making body owned by a group of stalkers, which pools their perception, distributes enemies and corpse reactions among them, and plans at the squad level rather than the individual level.

**Needs** — [`agent_manager.h`](agent_manager.h.md) · [`agent_corpse_manager.h`](agent_corpse_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`agent_explosive_manager.h`](agent_explosive_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`agent_manager_planner.h`](agent_manager_planner.h.md)
**Used by** — [`agent_manager.h`](agent_manager.h.md)
**Tier floor** — T3: a composition root and a fixed-order update; nothing here touches a device or a byte layout.

## Purpose

A **squad** — in this engine, a set of stalkers that share a group identifier — needs
decisions no single member can make: which member takes which enemy, who reacts to the
grenade, who picks up the item, whose death the others should respond to. This file owns
the object that makes them. It is a composition root: it builds six subordinate managers
plus a planner, drives them in a fixed order on a slow clock, and offers one entry point
for tearing an object's references out of all of them at once.

It is a separate file from its parts because each part (corpses, enemies, explosives,
locations, membership, memory) is independently sized and independently interesting; this
file holds only the assembly and the ordering.

## State

```text
RECORD AgentManager
  corpse       : CorpseManager        # who reacts to which dead body
  enemy        : EnemyManager         # enemy-to-member assignment
  explosive    : ExplosiveManager      # grenade / explosive avoidance
  location     : LocationManager       # claimed combat positions
  member       : MemberManager         # the squad roster
  memory       : MemoryManager         # the pooled perception of all members
  brain        : ManagerPlanner        # squad-level goal/plan search
  last_update_time : int               # global clock reading of the last update
  update_rate      : int               # milliseconds between updates; 1000

# Invariant: the manager must not be destroyed while the roster is non-empty —
#   members hold a reference to it, so the last member's departure is what ends it.
# Invariant: update runs only with a non-empty roster; every subordinate manager
#   assumes at least one member exists.
```

## `AgentManager` construction

**Contract** — Builds all seven subordinates and hands the planner a back-reference to the
manager so its evaluators can reach the roster. Sets the update clock to one second. No
failure path: allocation is assumed to succeed.

**Notes** — The manager can be driven two ways, selected at build time: it can register
itself with the engine's **scheduler** (asking for a 1000 ms period at both ends of the
allowed range and declaring a scheduling *scale* of one half, which makes it a
low-priority customer), or it can be polled by its owner and rate-limit itself against the
global clock. The shipped configuration is the second. The decision that survives is: the
squad brain runs about once per second, not per frame, and it is explicitly *not* a
frame-rate-sensitive computation — every subordinate is written to tolerate a second of
staleness. A rebuild may pick either drive; the period is the load-bearing part.

## `remove_links`

**Contract** — Given an object that is about to be destroyed, erases every reference to it
held anywhere in the squad. Fans out to all seven subordinates. Must be called before the
object's memory is released, and is the squad's half of the engine-wide invariant that a
destroyed entity is unreferenced everywhere before it dies.

## `update`

**Contract** — Advances the squad by one decision step, at most once per `update_rate`
milliseconds of global time. Returns immediately if the clock has not advanced or the
interval has not elapsed. Not re-entrant.

```text
FUNCTION update()
  IF now <= last_update_time THEN RETURN      # clock did not advance this frame
  IF now - last_update_time < update_rate THEN RETURN
  last_update_time = now
  update_impl()

FUNCTION update_impl()
  # The order is load-bearing: each stage consumes what the previous produced.
  memory.update()      # pool every member's perception into one shared picture
  corpse.update()      # decide which bodies matter
  enemy.update()       # rank and partition the enemies over that picture
  explosive.update()   # find explosives threatening members
  location.update()    # release and re-claim combat positions
  member.update()      # refresh roster-derived aggregates
  brain.update()       # plan over the results and issue the squad's order
```

**Invariants** — The roster is non-empty on entry. The brain runs last because its
evaluators read state the other six just wrote; running it first would plan against a
picture one second old.

**Notes** — The guard against a non-advancing clock is not paranoia about time going
backwards: several updates can be requested within one rendered frame, and the squad must
collapse them into one.
