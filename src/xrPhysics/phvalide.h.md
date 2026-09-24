# src/xrPhysics/phvalide.h

> Declares the level-bounds sanity test and the diagnostic text that accompanies a failure.

**Needs** — [`phvalide.cpp`](phvalide.cpp.md) · [`xrPhysics.h`](xrPhysics.h.md)
**Used by** — [`actor_mp_client_export.cpp`](../xrGame/actor_mp_client_export.cpp.md) · [`actor_mp_client_import.cpp`](../xrGame/actor_mp_client_import.cpp.md) · [`actor_mp_server_export.cpp`](../xrGame/actor_mp_server_export.cpp.md) · [`actor_mp_server_import.cpp`](../xrGame/actor_mp_server_import.cpp.md) · [`IActivationShape.cpp`](IActivationShape.cpp.md) · [`Physics.h`](Physics.h.md) · [`PhysicsShell.cpp`](PhysicsShell.cpp.md) · [`phvalide.cpp`](phvalide.cpp.md)
**Tier floor** — T2: a bounds predicate and a message formatter.

## Purpose

Declares the surface implemented in [`phvalide.cpp`](phvalide.cpp.md).

- `valid_pos` — is this point inside the current level's playable bounds?
- `ph_boundaries` — those bounds.
- `dbg_valid_pos_string` — the failure text: the offending position, the bounds it violated, and a
  full dump of the object that holds it.
- A checked-assertion wrapper that pairs the two, so that every site which asserts a position also
  produces the dump. That pairing is the reason the file exists: an assertion that says only
  "position invalid" is nearly useless in a world of several hundred moving objects, and making the
  dump automatic is what stops people writing the cheap version.

The wrapper and the diagnostic text compile away entirely outside development builds. What must
survive a rebuild is the pairing, not the mechanism: wherever a position is checked, the check knows
how to name the object that produced it.
