# src/xrGame/alife_schedule_registry.h

> The round-robin that gives every offline entity an alife tick, a bounded number of entities per pass.

**Needs** — [`safe_map_iterator.h`](safe_map_iterator.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`alife_schedule_registry_inline.h`](alife_schedule_registry_inline.h.md) · [`ai_debug.h`](ai_debug.h.md)
**Used by** — [`alife_combat_manager.cpp`](alife_combat_manager.cpp.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`alife_group_abstract.cpp`](alife_group_abstract.cpp.md) · [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_schedule_registry.cpp`](alife_schedule_registry.cpp.md) · [`alife_schedule_registry_inline.h`](alife_schedule_registry_inline.h.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_base2.cpp`](alife_simulator_base2.cpp.md) · [`alife_surge_manager.cpp`](alife_surge_manager.cpp.md) · [`alife_switch_manager.cpp`](alife_switch_manager.cpp.md) · [`alife_trader_abstract.cpp`](alife_trader_abstract.cpp.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md)
**Tier floor** — T2: a cursor over a map with a per-pass budget

## Purpose

This is the offline half of the engine's update budgeting — the coarse counterpart to the
client-side scheduler described in the glossary. Every server object that can think while
offline is in this set, and each alife pass advances only a fixed number of them, picking
up where the previous pass left off. The world therefore advances at a rate proportional
to the budget, not to the number of entities, which is what makes an unbounded number of
offline creatures affordable.

Membership rules are in
[`alife_schedule_registry.cpp`](alife_schedule_registry.cpp.md); the pass itself is in
[`alife_schedule_registry_inline.h`](alife_schedule_registry_inline.h.md).

## State

```text
RECORD ScheduleRegistry
  objects            : map<EntityId, ref Schedulable>   # ordered by identifier
  cursor             : position in that map             # survives across passes
  cycle_count        : int (64-bit)                     # incremented once per pass
  objects_per_update : int                              # the per-pass budget; defaults to 1
```

Plus, on each scheduled object, a **schedule counter** recording the pass in which it was
last updated. That field lives on the entity rather than here, which is what lets the
guard below be a single comparison.

Invariants:

- the cursor is always valid: adding to an empty set points it at the new entry, and
  removing the entry it points at advances it first;
- no object is updated twice in one pass, even if the cursor wraps all the way round —
  that is what the schedule counter is for;
- the default budget is **one object per pass**. It is raised at run time by the alife
  update manager according to how far behind the simulation is.

## The per-pass budget and the wrap guard

The pass walks forward from the cursor and stops on the first of two conditions: the
budget is exhausted, or the object under the cursor has already been updated in this pass.
The second is not a redundant safety net — with a small set and a large budget the cursor
wraps within a single pass, and without the guard a handful of entities would be advanced
several times while the rest of the world waited. The counter is compared against the pass
number rather than being a flag, so it never needs clearing.

The 64-bit pass counter is chosen so that it cannot wrap in any plausible session; a
rebuild using a narrower counter must handle the wrap, because a wrapped counter matching
a stale value on an entity would silently skip that entity forever.

## Exported units

- `CALifeScheduleRegistry` — construct with a budget of one.
- `add` / `remove` — membership, with the two qualification rules.
- `update` — run one pass.
- `object` — resolve an identifier to a scheduled object.
- `objects_per_update` — read and write the per-pass budget.

**Notes** — the underlying container is a general round-robin that also supports a
wall-clock time limit per pass; this registry deliberately turns that off and budgets by
*count* alone. Offline updates are cheap and uniform enough that a count is a good proxy,
and a count keeps the simulation's rate reproducible, which a wall-clock budget would not.
That reproducibility matters: conformance criterion 8 asks for a deterministic
simulation, and a time-limited offline scheduler would break it.
