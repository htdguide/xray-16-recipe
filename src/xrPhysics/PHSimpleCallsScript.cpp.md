# src/xrPhysics/PHSimpleCallsScript.cpp

> The three names and eight methods by which a script builds a physics timer or a thruster — frozen, because the shipped scripts use them.

**Needs** — [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`PHSimpleCalls.h`](PHSimpleCalls.h.md)
**Tier floor** — T3: a declaration of names against a script runtime.

## Purpose

The physics module's whole script surface, in one file. It is a separate file from the types it
exports because the binding layer is a seam — see
[the script binding seam](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) — and a rebuild
replacing that seam rewrites this file and nothing else.

## the exported surface

**Contract** — three script-visible classes. Every name below is **frozen**: the shipped scripts
name them, and criterion 10 of the
[conformance list](../../SYSTEM-REQUIREMENTS.md#6-conformance) requires them to load unmodified.

```text
CLASS "phcondition_callonstep"          # the step timer
  construct()                            # records the current step
  set_step(absolute_step)
  set_steps_interval(steps)
  set_time_interval_ms(milliseconds)
  set_time_interval_s(seconds)
  set_global_time_ms(milliseconds)
  set_global_time_s(seconds)

CLASS "phcondition_expireonstep" EXTENDS "phcondition_callonstep"
  construct()                            # fires at once, expires at its step

CLASS "phaction_constforce"
  construct(object, force_vector)
```

**Invariants** — the inheritance is declared to the script runtime, not merely implemented: a script
holding an expiry condition can call every setter the step condition declares, and the shipped
scripts do.

**Notes** — the four time setters are two overloaded pairs in the engine, distinguished by whether
the argument is a whole number of milliseconds or a fractional number of seconds. A script runtime
that does not distinguish integers from fractions cannot resolve that overload, which is why each
pair is exported under **two different names** with an explicit `_ms` / `_s` suffix. That is the
general rule for this seam and a rebuild inherits it: where the engine overloads on numeric width,
the script surface must name the variants apart.

There is no way for a script to construct the *bare* condition and action types from
[`PHScriptCall.h`](PHScriptCall.h.md) — those are built by the game layer when a script passes a
function to a physics call. What a script builds directly is only what is listed here.
