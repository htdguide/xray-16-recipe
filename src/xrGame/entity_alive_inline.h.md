# src/xrGame/entity_alive_inline.h

> Field access for the living-entity base: the condition model, and two behaviour flags.

**Needs** — [`entity_alive.h`](entity_alive.h.md) · [`EntityCondition.h`](EntityCondition.h.md)
**Used by** — [`entity_alive.cpp`](entity_alive.cpp.md) · [`entity_alive.h`](entity_alive.h.md)
**Tier floor** — T3: field access

## Purpose

Accessors for [`entity_alive.cpp`](entity_alive.cpp.md), split out so they are visible at
every call site. A rebuild merges this away entirely.

## State

`Stateless.`

## `conditions`

**Contract** — the creature's condition model, by reference. Asserted present: every living
entity has one from construction onward, so callers never test. This is the single busiest
accessor in the creature code — health, bleeding, radiation, wounds and immunities all reach
through it.

## `is_agresive` / `is_start_attack`

**Contract** — read and write two behaviour flags the planner sets and reads. Nothing in the
living-entity base consults either; they are storage on behalf of the layers above, placed
here because they must survive a creature changing behaviour state.
