# src/xrPhysics/PHSimpleCalls.cpp

> Converting a script's notion of "in three seconds" into the only clock that is the same on every machine: the physics world's step count.

**Needs** — [`PHSimpleCalls.h`](PHSimpleCalls.h.md) · [`PHWorld.h`](PHWorld.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md) · [`PhysicsCommon.h`](PhysicsCommon.h.md) · [`xrEngine/device.h`](../xrEngine/device.h.md)
**Used by** — [`PHSimpleCalls.h`](PHSimpleCalls.h.md)
**Tier floor** — T3: arithmetic over a step counter and one force application.

## Purpose

Short, and the whole content is the time-to-steps conversion and its rounding.

## the step condition

**Contract** — construction records the world's current step, or zero if no physics world exists yet.
The condition is true, and simultaneously obsolete, once the world's step count has passed the
recorded step — strictly past, not equal.

```text
FUNCTION is_true()  -> bool = world.steps_num > this.step
FUNCTION obsolete() -> bool = world.steps_num > this.step      # the same test, deliberately
```

**Notes** — building the condition before a physics world exists is legal and yields a condition that
is already satisfied, so the action runs on the first step of the first level. That is a reasonable
default for a script registered at load time and it is not obviously intended.

### setting the delay

```text
FUNCTION set_steps_interval(n)      = step := world.steps_num + n
FUNCTION set_time_interval(seconds) = set_steps_interval(CEILING(seconds / fixed_step))
FUNCTION set_time_interval(millis)  = set_time_interval(millis / 1000)
```

**Invariants** — the conversion rounds **up**. A delay shorter than one step still costs one step, so
a script can never ask for something to happen sooner than the next step, and no request is ever
silently satisfied in zero time.

### against the game clock

```text
FUNCTION set_global_time(t)
  interval := current_global_time - t
  IF interval < 0 THEN step := world.steps_num
  set_time_interval(interval)
```

**Notes** — this is broken in two ways and the shipped scripts work around it.

*The sign is inverted.* The interval is computed as *now minus the requested time*, so asking for a
moment in the future gives a negative interval and asking for a moment in the past gives a positive
one. Read literally, the call means "fire this as long after now as the given time was before now",
which is a plausible reading of "at a global time" only if the argument is a duration already
elapsed.

*The negative guard does not guard.* When the interval is negative the recorded step is set to the
current step — and then the conversion runs anyway and overwrites it, with a rounded-up negative
count, yielding a step already in the past. The condition is therefore immediately true. The
early-return that the guard clearly wanted is missing.

The net effect is that a future global time fires on the next step. A rebuild should decide what the
call means — almost certainly "fire when the game clock reaches this value" — and implement that;
nothing in the shipped scripts can be depending on the current behaviour beyond "it fires soon".

## `CPHShellBasedAction`

**Contract** — construction asserts the object is present and active. The action reports itself
obsolete as soon as the object is absent or has gone inactive.

**Notes** — the check is on every obsolescence query, not once, which is what makes it safe: the
object can deactivate at any point between registration and firing.

## `CPHConstForceAction`

**Contract** — applies the stored force vector to the whole object. One step's worth, because the
action retires when it runs.
