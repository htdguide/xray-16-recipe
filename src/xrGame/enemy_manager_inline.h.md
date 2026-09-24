# src/xrGame/enemy_manager_inline.h

> Accessors for the enemy manager, and the one rule among them that is not an accessor: a forced enemy overrides selection entirely.

**Needs** — [`enemy_manager.h`](enemy_manager.h.md) · [`entity_alive.h`](entity_alive.h.md)
**Used by** — [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`enemy_manager.h`](enemy_manager.h.md)
**Tier floor** — T3: field access with one override rule

## Purpose

Mostly field access for [`enemy_manager.cpp`](enemy_manager.cpp.md), split out so the
definitions are visible at every call site. A rebuild merges it away — except for the
selection override, which is a real decision and happens to live here.

## State

`Stateless.` It reads and writes the enemy manager's fields.

## `selected`

**Contract** — the creature's current enemy. Returns the **forced** enemy if one is set and
still alive; otherwise the inherited scored selection.

```text
FUNCTION selected() -> optional<EntityAlive>
  IF forced_enemy exists AND forced_enemy.alive: RETURN forced_enemy
  RETURN base.selected
```

**Notes** — this is the whole mechanism by which a smart cover tells its occupant who to
shoot at. Placing the override in the *reader* rather than in the selection pass means the
scored selection still runs every update underneath, keeping the last-enemy history and the
autosave gate correct; the creature simply reports a different answer. When the forced enemy
dies, the override lapses on its own and the scored selection is back, with no cleanup step.

A rebuild that short-circuits the selection pass instead would save a little work and break
both of those properties.

## `set_enemy` / `invalidate_enemy`

**Contract** — install and clear the forced enemy. Installing requires a living entity;
neither is optional and neither may be dead, because the override is meant to be a
deliberate instruction and a silently-ignored one would be very hard to diagnose. Clearing
always succeeds.

## `last_enemy` / `last_enemy_time`

**Contract** — the enemy the creature had at the last update that had one, and the global
clock reading at that update. Together they answer "how long have I been without an enemy",
which is what ends a combat state.

## `enable_enemy_change`

**Contract** — read and write the pin that stops loss of sight from reopening the enemy
decision. Does not prevent a change for any other reason — the enemy dying, relations
changing, or being hit by someone else all still reopen it.

## `useful_callback`

**Contract** — the script veto, exposed as a mutable reference so script binding can install
or clear it in place.
