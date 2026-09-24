# src/xrPhysics/PHScriptCall.h

> The eight ways a script can hand the physics world a condition to test and an action to run when it holds — and the rules by which two such requests count as the same one.

**Needs** — [`PHScriptCall.cpp`](PHScriptCall.cpp.md) · [`PHReqComparer.h`](PHReqComparer.h.md) · [`PHCommander.h`](PHCommander.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_game_object_use.cpp`](../xrGame/script_game_object_use.cpp.md) · [`PHReqComparer.h`](PHReqComparer.h.md) · [`PHScriptCall.cpp`](PHScriptCall.cpp.md)
**Tier floor** — T2: it holds references into a script runtime's value space and must release them deterministically.

## Purpose

The physics world runs at a fixed step and the script layer does not. A script that wants something
to happen *when a physical condition becomes true* cannot poll — it would have to run every step —
so it registers a standing request instead: a condition object the world tests each step, paired
with an action the world runs once the condition holds. The registry is
[`PHCommander.h`](PHCommander.h.md); this file is the script-facing half of it.

Eight types appear here, and they are eight only because there are three independent choices:
condition or action; a bare function or a method on a script object; and whether a game object's
lifetime bounds the request. A rebuild collapses them to one type with a callable and an optional
owner.

## the condition/action protocol

**Contract** — what the world asks of everything registered with it.

```text
INTERFACE Condition
  FUNCTION is_true() -> bool         # tested once per step while the request stands
  FUNCTION obsolete() -> bool        # true when this request can never fire again

INTERFACE Action
  FUNCTION run()                     # called once, when the paired condition holds
  FUNCTION obsolete() -> bool        # true when this request has been consumed
```

**Invariants** — the two obsolescence rules are opposites and both are load-bearing:

- **A condition is never obsolete.** It may be false forever without being retired.
- **An action becomes obsolete the moment it runs.** Every action is one-shot.

Together these say: a standing request fires exactly once and is then removed. A script wanting a
repeating effect re-registers. The exception is the game-object-bounded condition, which reports
itself obsolete once it has been *true* — retiring the request when its purpose is served rather
than waiting for the action.

## the eight shapes

| | condition | action |
|---|---|---|
| a bare script function | `CPHScriptCondition` | `CPHScriptAction` |
| a named method on a script object | `CPHScriptObjectCondition` | `CPHScriptObjectAction` |
| a script callback bound to an object | `CPHScriptObjectConditionN` | `CPHScriptObjectActionN` |
| the above, owned by a game object | `CPHScriptGameObjectCondition` | `CPHScriptGameObjectAction` |

**Notes** — the middle two rows differ in *when the method is resolved*. The named-method form stores
an object and a method name and looks the method up at call time, so a script that replaces the
method afterwards changes what runs. The bound-callback form captures the function itself at
registration. Both exist in the shipped scripts; the difference is observable and must be preserved.

The game-object-owned pair adds an identity — a game object's numeric handle — that the world can
match on, so that "cancel every physics request belonging to this object" is expressible when the
object is destroyed. Without it, a destroyed object's request would run against a dangling
reference. This is the specific reason those two types exist.

## equality — when is this the same request?

**Contract** — each type answers whether another request is equivalent to it, and offers itself to a
comparer that is hunting for a particular kind (see [`PHReqComparer.h`](PHReqComparer.h.md)).

- Two **bare-function** requests match when they reference the same script function value.
- Two **named-method** requests match when the method names are equal **and** the script objects
  compare equal — using a comparison that treats two absent values as equal, since a script object
  may legitimately be nothing.
- Two **bound-callback** requests match when the captured callbacks are equal.
- Two **game-object** requests match when the owning objects have the same handle.

**Invariants** — the method name is compared *first* and by an interned-string identity rather than
by content, which is cheap and exact because the name came from the script's own constant table.

**Notes** — the "two absent values are equal" rule is not pedantry. A script object that has been
collected reads as absent, and two dead requests must still be recognizable as the same request so
the registry can drop them both.

## the two comparers

**`CPHSriptReqObjComparer`** — "find every request bound to *this* script object", regardless of
which of the four object-bearing shapes it takes. Holds a copy of the script object.

**`CPHSriptReqGObjComparer`** — "find every request owned by *this* game object". Holds the game
object directly, and matches only the two game-object shapes.

**Notes** — these are the two questions the game actually asks, and they are why the comparison is
dispatched rather than being a method on the request: the asker knows what it is looking for, the
request knows what it is, and only the pair determines the answer.

## Notes

Every one of these types owns a reference into the script runtime's value space and must release it
when it dies, or the script runtime's collector never frees the function. The reference is held
indirectly rather than by value because the script binding's value type cannot be default-built and
the registry needs to store these uniformly. In a rebuild where script values are ordinary
reference-counted handles, this indirection and every copy constructor beside it disappears — what
must survive is that the handle is released deterministically when the request is retired, not at
some later collection.

A privately-declared copy constructor on the bare condition means it is created in place and never
copied, while every other type here is copyable. The distinction has no discoverable reason and a
rebuild should make them uniform.
