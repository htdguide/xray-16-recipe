# src/xrGame/member_enemy_inline.h

> Construction, equality and ordering for a squad's shared enemy entry.

**Needs** — [`member_enemy.h`](member_enemy.h.md)
**Used by** — [`member_enemy.h`](member_enemy.h.md)
**Tier floor** — T3: field writes and two comparisons

## Purpose

Carries the shared enemy entry's bodies out of the declaration. A rebuild folds them in; the
substance is in [`member_enemy.h`](member_enemy.h.md).

## State

`Stateless.`

## Construction · equality · ordering

**Contract** — construction records the enemy and the reporting member's bit, sets confidence
to full, clears the assignment mask, and zeroes the observation time. Equality compares only
the enemy. Ordering places **higher confidence first** — the comparison is deliberately
reversed so that a plain ascending sort yields descending confidence.
