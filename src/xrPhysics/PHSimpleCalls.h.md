# src/xrPhysics/PHSimpleCalls.h

> The two standing requests the engine provides ready-made: "do this after N steps" and "push this object with a constant force" — the timer and the thruster a script builds everything else from.

**Needs** — [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md) · [`PHSimpleCallsScript.cpp`](PHSimpleCallsScript.cpp.md) · [`PHCommander.h`](PHCommander.h.md) · [`PHReqComparer.h`](PHReqComparer.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`PHCommander.cpp`](PHCommander.cpp.md) · [`PHCommander.h`](PHCommander.h.md) · [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md) · [`PHSimpleCallsScript.cpp`](PHSimpleCallsScript.cpp.md) · [`PHWorld.cpp`](PHWorld.cpp.md)
**Tier floor** — T2: it is step counting and a force application, exported to a script runtime.

## Purpose

[`PHScriptCall.h`](PHScriptCall.h.md) lets a script supply its own condition and action. This file
supplies two that the engine implements natively, because they are what scripts overwhelmingly
want and because writing them in script would mean a script call every step.

## `CPHCallOnStepCondition`

**Contract** — a condition that becomes true once the world's step counter passes a recorded step,
and is obsolete at the same moment. Constructing it records the current step, so an unconfigured
one fires on the next step.

```text
RECORD CallOnStepCondition
  step : int (64-bit)      # the step this condition waits for
```

**Invariants** — `is_true` and `obsolete` are the *same test*. A step condition is spent at the
moment it fires; it never lingers.

**Exported units** — `set_step` (an absolute step), `set_steps_interval` (this many steps from now),
`set_time_interval` in seconds or milliseconds, `set_global_time` against the game's own clock.

**Notes** — expressing a delay in *steps* rather than seconds is the decision. The physics world
advances by a fixed timestep and its step counter is the only clock that is identical on every
machine, so a delay measured in steps satisfies the
[determinism requirement](../../SYSTEM-REQUIREMENTS.md#6-conformance) while a delay measured in
wall-clock seconds does not. The time-based setters exist for script convenience and convert to
steps at once — see [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md) for how, and for the rounding rule.

## `CPHExpireOnStepCondition`

**Contract** — the same thing with `is_true` forced to yes: it fires immediately and retires at its
recorded step regardless.

**Notes** — the pair is the difference between "run this in three seconds" and "run this now, and
stop offering to run it in three seconds". The second is how a script registers something whose
*expiry* matters and whose triggering does not — the action runs on the next step, and the standing
request is guaranteed gone by the recorded step even if the action never ran.

## `CPHShellBasedAction`

**Contract** — an action tied to one physical object, which becomes obsolete as soon as that object
stops being active. Requires an active object at construction.

**Invariants** — this is the lifetime rule that keeps standing requests from reaching into
deactivated physics. A rebuild must have an equivalent: the registry outlives objects, and an action
holding a dead one is the classic crash.

## `CPHConstForceAction`

**Contract** — applies a fixed force vector to a whole object each time it runs. Comparable by which
object it acts on.

**Notes** — because actions are one-shot (see [`PHScriptCall.h`](PHScriptCall.h.md)), a "constant"
force is constant only for one step; a script that wants a sustained push re-registers the pair each
time, or pairs it with a step condition that re-arms. That is a slightly surprising reading of the
name and it is what the shipped scripts do.

## `CPHReqComparerHasShell`

**Contract** — the question "is there already a constant-force request acting on this object?",
asked so a script that repeatedly declares a thruster does not stack a dozen of them. See
[`PHReqComparer.h`](PHReqComparer.h.md) for the dispatch.

## Notes

All three script-visible types are exported to the script runtime under names that are part of the
frozen script surface: the step condition, the expiry condition and the constant force. Their
registration is in [`PHSimpleCallsScript.cpp`](PHSimpleCallsScript.cpp.md), and the names and
signatures there are a compatibility constraint, not a choice.
