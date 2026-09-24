# src/xrGame/hit_memory_manager_inline.h

> The hit memory's cheap accessors and its construction.

**Needs** — [`hit_memory_manager.h`](hit_memory_manager.h.md)
**Used by** — [`hit_memory_manager.h`](hit_memory_manager.h.md)
**Tier floor** — T2: accessors

## Purpose

The accessors that run inside the per-frame perception loop, separated so they inline. The
split is incidental; a rebuild may fold them in.

## State

`Stateless` — it defines the construction and reads of [`hit_memory_manager.h`](hit_memory_manager.h.md)'s record.

## `CHitMemoryManager` (construction)

**Contract** — binds the manager to its creature, which must exist, and to the same creature
viewed as a stalker, which may be absent. **The hit list itself is deliberately left
unset** — construction does not decide where the memory lives.

**Invariants** — the list is supplied afterwards, by the reinitialization (which clears it to
none) and then by the group registration (which points it at a shared list) or by the
creature's own memory manager. Until then every query asserts. That is the whole reason
group perception can exist: the manager never owns its storage.

## `objects`

**Contract** — the hit list. Asserts the list has been bound.

## `set_squad_objects`

**Contract** — points this manager at a different list. Does not copy, migrate or free
anything: whatever was recorded in the old list stays there.

**Invariants** — this is called on joining and leaving a group, and the consequence is that a
creature *loses its personal hit memory* when it joins a group and *loses the group's* when
it leaves. Neither transition carries memory across. A rebuild must accept that as the
behaviour, because it is what makes a stalker forget a private grudge on joining a squad.

## `object`, `last_hit_object_id`, `last_hit_time`

**Contract** — the served creature (asserted present), and the two scalars recording the most
recent attacker and when. Those two are *not* part of the shared list: they are per-creature
even inside a group, because "who shot me last" is personal.
