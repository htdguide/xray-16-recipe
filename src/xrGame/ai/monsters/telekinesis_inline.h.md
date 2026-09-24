# src/xrGame/ai/monsters/telekinesis_inline.h

> A superseded, template-based telekinesis controller with a different interface from the one that ships. Included by nothing; it is dead source that the build description still lists.

**Needs** — _(none that resolve: it names a class shape that no longer exists)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it would be, if it were reachable

## Purpose

This file implements a telekinesis controller that is **not** the one in
[`telekinesis.h`](telekinesis.h.md) / [`telekinesis.cpp`](telekinesis.cpp.md). It is an
older generation: the controller is a template over its owner type, it stores held objects
*by value* rather than by owned pointer, it has no per-object allocator hook, and its
public names differ throughout — an external initialiser that takes strength, height and
hold duration up front for *all* objects, an activate that sweeps a fixed 10-metre radius
and grabs everything with a physics body at once, a throw that fires all objects and
deactivates, and a scheduled update carrying the phase machine inline.

**It is included by no file.** The shipped controller's header does not pull it in, and
nothing else names it. The build description lists it as a header, which is why it survives
in the tree at all. A rebuilder should delete it.

It is documented for one reason: it records a design the shipped version deliberately moved
away from, and the contrast is instructive.

| Superseded design (this file) | Shipped design |
|---|---|
| Grab everything within a fixed 10-metre sphere on activation | Owner nominates each object individually |
| One strength, height and hold duration for the whole controller | Per object, supplied at activation |
| Objects stored by value; no polymorphism | Objects owned by pointer, with an allocator hook an owner overrides |
| Rising force scaled by the physics step | Rising force independent of the step (see [`telekinetic_object.cpp`](telekinetic_object.cpp.md)) |
| No thrown phase — throwing deactivates immediately | A thrown phase with its own expiry |

The move from "grab a sphere" to "nominate objects" is the one that matters: it is what
lets the burer pick a specific piece of debris to hurl at the player instead of lifting the
whole room.

## Exported units

None reachable.
