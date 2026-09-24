# src/xrGame/script_object_action.h

> Declares the object channel of a scripted action: what an entity should do with a thing it is holding, or with one of its own bones.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_object_action.cpp`](script_object_action.cpp.md) · [`script_object_action_inline.h`](script_object_action_inline.h.md) · [`script_object_action_script.cpp`](script_object_action_script.cpp.md)
**Tier floor** — T2: a plain record

## Purpose

The channel that orders *item handling*: draw the weapon, holster it, reload it, fire a
burst of this many rounds, pick that up, drop it, switch the torch on. It is aimed either
at an inventory item, named by object, or at a place on the entity's own skeleton, named by
bone — which is how a script says "aim at your left hand" or drives an object mounted on a
bone.

## State

```text
RECORD ScriptObjectAction EXTENDS ActionChannel   # the base supplies `completed`
  object     : optional<ClientObject>  # the item this order concerns
  bone_name  : text                    # the alternative target: a bone on the actor itself
  action     : enum { idle, show, hide, take, drop, strapped,
                      aim1, aim2, reload1, reload2, fire1, fire2,
                      switch1, switch2, activate, deactivate, use,
                      turn_on, turn_off }
  queue_size : int                     # rounds per burst for the firing actions;
                                       # the all-ones sentinel means "no limit"
```

**Invariants**

- `object` and `bone_name` are alternatives, and setting one does not clear the other. A
  channel that has had both set is not rejected; the consumer reads whichever its action
  kind needs. A rebuild should treat this as a two-case choice and would be right to make
  it one.
- `action` defaults to *idle*, which orders nothing, so a default-constructed channel is
  inert.
- `queue_size` is only consulted by the firing actions. Its default is the all-ones value
  meaning unlimited, not zero — zero would mean "fire nothing" and is a distinct, usable
  order.
- Every setter clears the completion flag, so writing an unchanged value reopens a finished
  channel. See [`script_object_action_inline.h`](script_object_action_inline.h.md).

## Exported units

- construct — inert.
- construct from (object, action, burst size) — the common form.
- construct from (bone name, action) — the bone-targeted form.
- construct from (action) — an order concerning no particular thing, such as *hide*.
- `set_object(game_object)` and `set_object(bone_name)` — the two ways to name a target.
- `set_object_action(action)`, `set_queue_size(count)`.
- `initialize` — does nothing; the channel has no per-execution state.

**Notes**

The numbered pairs — `aim1`/`aim2`, `reload1`/`reload2`, `fire1`/`fire2`, `switch1`/
`switch2` — address a weapon's **two firing modes**: primary and the under-barrel or
alternate one. They are not two levels of the same thing. A rebuild should keep the
numbering because shipped scripts and shipped animations agree on it.
