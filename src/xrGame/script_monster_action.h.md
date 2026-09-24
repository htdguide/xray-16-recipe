# src/xrGame/script_monster_action.h

> Declares the monster-behaviour channel of a scripted action: a coarse global behaviour plus an optional target.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_monster_action.cpp`](script_monster_action.cpp.md) · [`script_monster_action_inline.h`](script_monster_action_inline.h.md) · [`script_monster_action_script.cpp`](script_monster_action_script.cpp.md)
**Tier floor** — T2: a plain record over an action channel

## Purpose

One of the channels a [scripted action](script_entity_action.h.md) carries. Where the other
channels describe *how* an entity moves, looks, animates or sounds, this one names a
coarse **global behaviour** for a monster — rest, eat, attack, panic — and leaves the
monster's own brain to decide the details. It is the escape hatch that lets a script steer
a creature without scripting its every step.

## State

```text
RECORD ScriptMonsterAction EXTENDS ActionChannel   # the base supplies `completed`
  action : enum { none, rest, eat, attack, panic }
  target : optional<ClientObject>   # what to attack or eat; unused by rest and panic
```

**Invariants**

- `action` defaults to `none`, which the queue pump reads as "this channel imposes
  nothing" — an action carrying a movement channel and no monster channel leaves the
  monster's global behaviour alone.
- `target` is stored as the **client object**, not the script facade, even though it is
  always set from a facade. The channel outlives the script call that built it and the
  engine consumes it from the scheduler, where only the live instance is meaningful.
- The channel's `completed` flag lives in the shared action-channel base and is cleared by
  every constructor that names a behaviour; the default constructor leaves it as the base
  set it.

## Exported units

- construct — an empty channel imposing nothing.
- construct from a behaviour — sets the behaviour and marks the channel unfinished.
- construct from a behaviour and a target — the same, plus the target.
- `set_object(game_object)` — replaces the target, unwrapping the script facade to the
  client object it fronts.

**Notes**

The behaviour set is deliberately tiny. The full monster brain has dozens of states; only
these four are addressable from a script, because they are the four that make sense to
*impose* from outside without knowing what the creature is currently doing.
