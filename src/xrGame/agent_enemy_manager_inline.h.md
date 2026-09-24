# src/xrGame/agent_enemy_manager_inline.h

> Construction and accessors for the squad's target assignment.

**Needs** — [`agent_enemy_manager.h`](agent_enemy_manager.h.md)
**Used by** — [`agent_enemy_manager.h`](agent_enemy_manager.h.md)
**Tier floor** — T3: three accessors

## Purpose

The trivial half of the enemy manager; everything that decides anything is in
[`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md).

## State

Adds nothing.

## Construction

**Contract** — binds the squad brain, permanently; requires it to be present. Starts with
both pool-classification flags false, which is the correct initial reading of an empty
pool: there is no wounded enemy and it is not true that only wounded enemies remain.

## `object` / `enemies`

**Contract** — the squad brain (binding required) and the pooled enemy list, writable. The
list is published because the location manager reads enemy positions from it when deciding
whether a cover point is too close to a known enemy.
