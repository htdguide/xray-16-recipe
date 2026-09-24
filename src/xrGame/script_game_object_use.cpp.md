# src/xrGame/script_game_object_use.cpp

> The facade's construction and destruction, its identity and death, faction relations, the callback installation surface, and the two entry points that let a script push on the physics world.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`GameObject.h`](GameObject.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`searchlight.h`](searchlight.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`doors_manager.h`](doors_manager.h.md) · [`xrPhysics/PHCommander.h`](../xrPhysics/PHCommander.h.md) · [`xrPhysics/PHScriptCall.h`](../xrPhysics/PHScriptCall.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Rigid-body physics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation, plus two deferred-call registrations against the physics seam

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across,
and the one that carries the facade's own lifetime. Everything here follows the
[guarded-delegation pattern](script_game_object.cpp.md#the-guarded-delegation-pattern);
what is worth writing down is the handful of places that depart from it.

## Facade lifetime

**Contract** — a facade is constructed against a client object and **fails hard** if given
none: a facade with nothing behind it is not a degraded facade, it is a bug in the engine
rather than in a script, and the assertion is the only one of its kind in the nine files.
Destruction unregisters the facade's door entry if it has one, and does nothing otherwise.

**Invariants**

- The facade holds its client object for its whole life and never rebinds. Identity is the
  client object's.
- The **door registration is the facade's only owned resource**, and it is owned here
  rather than by the client object because scripts, not the engine, decide which physics
  objects count as doors for the navigation layer. Dropping it on destruction is what stops
  a stale volume blocking pathfinding after a level reload.

## Identity

**Contract** — `parent` returns the facade of the client object's container, or nothing at
the top level; `class_id`, `name` and `section` read the entity's class tag, its unique
instance name and its configuration section. All four are unconditional: every client
object has them.

**Notes**

`parent` returns the *other object's own facade*, not a new one. Facades are one per client
object and looked up, never constructed, outside the object factory. A rebuild that mints a
facade per call breaks script identity comparison, which shipped scripts rely on.

## `kill`

**Contract** — kills a living entity, blaming the given killer or the victim itself when
none is named. Three guards, in this order:

```text
FUNCTION kill(killer, bypass_actor_check)
  IF god mode is on AND this entity is the actor THEN RETURN   # silently
  IF this entity cannot die THEN log script error; RETURN
  IF this entity is already dead THEN
    log script error "attempt to kill dead object <name>"
    RETURN
  entity.die(blamed_on = killer's id, or this entity's own id, bypass_actor_check)
```

**Invariants**

- The god-mode check comes **first**, before the check that the entity can die at all, and
  it is silent. A script killing the player under god mode must be a no-op, not an error —
  shipped scripts kill the actor in scripted deaths and the player's cheat must win.
- Killing an already-dead entity is reported rather than ignored, because it is almost
  always a script that lost track of state, and dying twice would re-run the death
  callbacks.
- Blaming the victim itself when no killer is named is what makes an unattributed scripted
  death count as a suicide rather than as a kill by entity zero.

The `bypass_actor_check` flag reaches the death path unmodified; it exists for scripted
deaths that must not go through the actor's own death arbitration.

## `alive`

**Contract** — whether a living entity is alive. Anything that cannot die answers **false**
with a script error, not true.

## `relation_to(other)`

**Contract** — the faction relation from this entity to another: friend, neutral, enemy.
Both sides must be living entities, and the two failures are reported *differently* — one
names this object, the other names the other — because a script passing the wrong argument
and a script calling on the wrong object are different mistakes. Either way the answer is
the "no such relation" marker rather than neutral, so a script can tell "no opinion" from
"not applicable".

## `action_planner`

**Contract** — the creature's planner, or nothing with a script error on anything that has
no brain. Only stalkers have one; asking a dog or a crate is the common script error this
guard exists for.

## Callbacks

**Contract** — the facade's callback surface is three overloads of one operation:
install a script function for a named event, install one bound to a script object so the
callback can be a method, or — with no function at all — **clear** that event's callback.
`clear_callbacks` drops all of them at once.

**Invariants**

- Installing replaces; there is no list. One script owns each event on each object, and a
  second installer silently displaces the first. That is a frequent source of mod conflicts
  and it is frozen behaviour.
- The event set is the enumeration in [`game_object_space.h`](game_object_space.h.md), and
  its names are frozen.
- `clear_callbacks` is what the engine calls when an object goes offline, so a script that
  installed a callback and did not remove it does not leak into the next session.

The enemy-selection callback is the same three-way shape but lives on the creature's memory
subsystem rather than in the object's callback table, because it is a *filter* consulted
during enemy selection rather than a notification. Its answer decides whether a remembered
enemy is worth considering at all.

## `set_fastcall`

**Contract** — registers a script predicate to be evaluated by the physics stepper rather
than by the frame loop, together with an action that does nothing. Any predicate previously
registered against this same object is removed first.

```text
FUNCTION set_fastcall(predicate, bound_object)
  condition = a physics-world condition that calls the script predicate
  action    = a no-op
  deferred: remove every registered call whose subject is this object
  deferred: add (condition, action)
```

**Invariants**

- Both the removal and the addition are **deferred** to the end of the current physics
  step. Registering a call from inside a call being iterated would invalidate the
  iteration; deferral is the standard fix and is load-bearing here because scripts
  routinely call this from inside a fastcall.
- One fastcall per object: registering a second replaces the first. The removal is
  unconditional, so re-registering the same predicate is the idiom for "restart it".
- The action is deliberately empty. The whole mechanism exists for the *predicate's* side
  effects — the script runs at physics rate, which is finer than the frame rate, and
  returns whether it wants to run again.

**Notes**

"Fastcall" is the engine's own word for a script callback attached to the physics step.
It is the only way a script can observe the world between frames, and it is what scripted
physics puzzles are built on. A rebuild that runs scripts only per frame cannot reproduce
them.

## `set_const_force`

**Contract** — applies a constant force in a direction to an object's rigid body for a
number of physics steps, then expires. Reports a script error and does nothing when there
is no physics world or the object has no rigid body.

```text
FUNCTION set_const_force(direction, magnitude, step_count)
  IF no physics world THEN log script error; RETURN
  IF object has no rigid body THEN log script error naming the object; RETURN
  register (expire-after step_count, apply force direction * magnitude) with the physics world
```

**Invariants** — the duration is counted in **physics steps**, not seconds, so the same
script produces the same impulse regardless of frame rate. A rebuild that measures it in
time changes every scripted physics effect in the game.

**Notes**

The physics-world check happens before the rigid-body check even though the rigid body is
fetched first. Fetching it on an object that has no physics role at all is itself unsafe in
the original; a rebuild should test the object's role before reaching for the body.

## Use-prompt text and searchlight aim

**Contract** — `set_tip_text`, `set_tip_text_default` and `set_nonscript_usable` control
what the player sees and whether the object responds to the use key at all, on any object.
`current_direction` reads a searchlight's present aim, answering the zero vector with a
script error on anything else.
