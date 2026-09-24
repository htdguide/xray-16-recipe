# src/xrPhysics/PHWorldScript.cpp

> Exposes the physics world to Lua: gravity, the global time factor, and the deferred-call hook.

**Needs** — [`PHWorld.h`](PHWorld.h.md) · [`PHCommander.h`](PHCommander.h.md) · [`Physics.h`](Physics.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a binding table; no data layout, no timing.

## Purpose

Stateless. Declares the script-visible surface of the physics world. Small, but it is part of the
strictest acceptance criterion in the project — every shipped script must run unmodified, which
fixes these names and signatures exactly.

## `script_register`

**Contract** — registers one class and three free functions into the script virtual machine.

```text
CLASS "physics_world"
  set_gravity(real)         # world gravity magnitude; positive, applied downward
  gravity() -> real
  add_call(condition, action)   # schedule a deferred call, checked once per simulation step

FUNCTION level.physics_world() -> physics_world        # the single world instance
FUNCTION level.get_ph_time_factor() -> real            # global simulation speed multiplier
FUNCTION level.set_ph_time_factor(real) -> real
```

**Notes** — the two time-factor functions are an extension over the original game's surface, not
part of the shipped-script contract; they are marked as such in the source. The time factor scales
elapsed real time *before* the accumulator sees it (see [`PHWorld.cpp`](PHWorld.cpp.md)), so it
changes how many fixed steps a frame produces, not the step size — slow motion stays deterministic.

`add_call` is the entry point for the whole condition/action machinery in
[`PHScriptCall.cpp`](PHScriptCall.cpp.md) and [`PHSimpleCalls.cpp`](PHSimpleCalls.cpp.md).
