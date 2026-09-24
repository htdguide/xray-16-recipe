# src/xrPhysics/PHCommander.h

> The condition/action list physics evaluates once per step — the engine's way of
> saying "when this becomes true, do that" without anybody polling.

**Needs** — [`PHCommander.cpp`](PHCommander.cpp.md) · [`PHReqComparer.h`](PHReqComparer.h.md) · [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md)
**Used by** — [`Level.cpp`](../xrGame/Level.cpp.md) · [`PhysicsShellHolder.cpp`](../xrGame/PhysicsShellHolder.cpp.md) · [`level_script.cpp`](../xrGame/level_script.cpp.md) · [`physics_game.cpp`](../xrGame/physics_game.cpp.md) · [`script_game_object_use.cpp`](../xrGame/script_game_object_use.cpp.md) · [`IPHWorld.h`](IPHWorld.h.md) · [`PHCommander.cpp`](PHCommander.cpp.md) · [`PHScriptCall.cpp`](PHScriptCall.cpp.md) · [`PHScriptCall.h`](PHScriptCall.h.md) · [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`PHWorld.cpp`](PHWorld.cpp.md) · [`PHWorldScript.cpp`](PHWorldScript.cpp.md)
**Tier floor** — T3: a list of closures evaluated in order. Only the lock and the
re-entrancy rules keep it from being trivial.

## Purpose

Physics runs on a fixed timestep of its own, faster than the frame and possibly on another
thread, and many things in the game need to happen *at* a step rather than at a frame: apply
a constant force for the next two hundred steps, wake this object when that one stops, play
a splash when a body enters water, call a script function when a joint breaks. Polling from
the game layer would sample the wrong steps.

The commander is the answer: a flat list of **calls**, each a condition paired with an
action, evaluated in order once per step. Both halves are supplied by the caller, so the
mechanism knows nothing about what it is scheduling — the physics module, the game layer
and Lua all put calls on the same list.

This header is substantive despite having a companion `.cpp`, because the three abstract
types it declares *are* the contract a rebuild must satisfy; the `.cpp` holds only the list
management.

## `CPHReqBase`

**Contract** — the root of both halves. Two obligations:

- **`obsolete`** — "I will never be useful again." Asked after every evaluation; a call
  whose condition or action reports obsolete is removed and destroyed. This is the only
  lifetime mechanism: there are no handles, no cancellation tokens and no reference counts.
  A call's author decides when its work is done by answering this question.
- **`compare`** — "are you the thing this visitor is looking for?" Used to find and remove
  calls without holding a pointer to them; see the comparer discussion below.

**Invariants** — obsolescence is checked *after* the action runs, so a one-shot call always
fires exactly once. A condition that becomes obsolete by being evaluated must still return
its verdict for that evaluation.

## `CPHCondition` and `CPHAction`

**Contract** — a condition answers `is_true` once per step; an action has `run`. Neither may
assume it is called on any particular thread, and both are invoked from inside the physics
step, so both may read and write body state directly — that is the point of the mechanism.

**Notes** — the two are separate types rather than one closure because they are matched
independently: a caller can ask "is there a call with *this* condition and *that* action"
and remove it, which is how a repeating effect is replaced rather than duplicated.

## `CPHOnesCondition` and `CPHDummiAction`

**Contract** — the two degenerate members. The one-shot condition returns true the first
time it is asked and reports itself obsolete forever after; the dummy action does nothing
and is never obsolete.

**Notes** — pairing the one-shot condition with a real action gives "do this at the next
step and forget it", which is the single most common use of the whole mechanism. Pairing a
real condition with the dummy action gives a call that exists only to be *found* by a
comparer — a marker on the list. Both idioms appear in the shipped code.

## `CPHCall`

**Contract** — owns one condition and one action and destroys both with itself. `check`
evaluates the condition and runs the action if it holds; `obsolete` is true when either half
is; `equal` matches a call against a pair of comparers; `is_any` matches it against one
comparer applied to either half.

**Invariants** — the call owns both halves outright. There is no way to detach or share
them, and the caller hands over ownership at the moment it adds the call.

## `CPHCommander`

**Contract** — the list. Its surface splits into three groups:

- **Adding** — plainly, thread-safely (under the lock), uniquely (only if no equal call is
  present), or *deferred* (queued and applied at the top of the next evaluation).
- **Removing** — by iterator, by a pair of comparers, by a single comparer matching either
  half, thread-safely, or deferred.
- **Evaluating** — `update` runs one pass; `update_threadsafety` does so under the lock;
  `clear` destroys everything.

**Invariants** — evaluation order is insertion order, and a rebuild must preserve it. Calls
routinely depend on each other within a step — a force applied by one call and read by
another — and the shipped content was tuned against this order.

**Notes** — the *comparers* deserve their own explanation, because they look like an
elaborate way to compare pointers and are not. A caller usually wants to remove "every call
belonging to this object", and it does not hold the calls. A comparer is a visitor that each
condition and action is offered; the concrete types that care override the matching
overload and answer. The base answers "no" to everything, so a type that does not opt in is
simply never found — which is why removing calls for a destroyed object is safe even when
some of that object's calls are of types that were never taught to match. The concrete
comparers are declared in [`PHReqComparer.h`](PHReqComparer.h.md).

The **deferred** add and remove exist because the list is mutated from inside its own
evaluation — an action may add or remove calls, including its own. See
[`PHCommander.cpp`](PHCommander.cpp.md) for how that is survived.
