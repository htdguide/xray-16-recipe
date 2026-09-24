# src/xrPhysics/PHScriptCall.cpp

> Calling into the script runtime from inside the physics step, and the one-shot rule that retires an action the moment it has run.

**Needs** — [`PHScriptCall.h`](PHScriptCall.h.md) · [`PHCommander.h`](PHCommander.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`PHScriptCall.h`](PHScriptCall.h.md)
**Tier floor** — T2: it calls across a script boundary and owns handles into the script runtime.

## Purpose

The bodies behind [`PHScriptCall.h`](PHScriptCall.h.md). There is very little decision here — the
substance of the design is in the header — but three things are worth stating.

## the one-shot rule

**Contract** — every action's `run` does two things in order: it calls the script, and it marks
itself obsolete. Every condition's `obsolete` answers no, unconditionally.

```text
FUNCTION run()
  invoke the script callable
  obsolete := true
```

**Invariants** — the mark is set *after* the call, not before. A script that raises during its action
therefore leaves the request live and it runs again next step. Whether that is intended is not
discoverable; it is the behaviour, and a repeating error in a physics action shows up as a
per-step error spam rather than a single message.

## the four call shapes

**Contract** — the four ways of reaching the script differ only in what is stored and how the call
is made.

```text
bare function       → invoke the stored callable with no arguments
named method        → look the method up on the stored object BY NAME, now, and call it
bound callback      → invoke the stored (object, function) pair
game-object bounded → as the bound callback, plus: a CONDITION records its own result as
                      its obsolescence, so a condition that has fired is retired
```

**Notes** — the named-method form resolving the method at call time is the interesting one: it means
a script can swap an object's handler after registering the request and the swap takes effect. The
shipped scripts use this.

The game-object condition's obsolescence rule inverts the normal one — a condition that has been
true is done — which pairs correctly with the action beside it: once the condition fires, the action
fires, the action marks itself spent, and the condition marks itself spent too, so the whole request
leaves the registry in the same pass. Without it the condition would keep testing true against a
retired action.

## lifetime

**Contract** — every type here holds a script-runtime reference for its whole life and releases it on
destruction. The copy constructors copy the reference rather than sharing it.

**Notes** — this is the incidental half of the file, and the problem it solves is that the registry
stores these by value while the script binding's value type cannot be stored uninitialized. A
rebuild whose script handles are ordinary reference-counted values deletes every constructor and
destructor in this file. What must survive is the release: a request that is dropped without
releasing its script reference pins a closure — and through the closure, potentially a whole level's
worth of script state — for the rest of the session.
