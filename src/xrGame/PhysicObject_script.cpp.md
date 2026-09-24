# src/xrGame/PhysicObject_script.cpp

> Exports the physics prop and its destructible variant to the script virtual machine.

**Needs** — [`PhysicObject.h`](PhysicObject.h.md) · [`DestroyablePhysicsObject.h`](DestroyablePhysicsObject.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Declares two classes to Lua: the physics prop as a subclass of the game object facade,
and the destructible prop as a subclass of the prop. Both are default-constructible from
script.

## State

`Stateless.`

## `CPhysicObject::script_register`

**Contract** — registers `CPhysicObject` (deriving from the game object facade) with a
no-argument constructor and nine methods, and `CDestroyablePhysicsObject` (deriving from
`CPhysicObject`) with a no-argument constructor and nothing else. Runs once at
script-engine bring-up. Names and signatures are frozen by conformance criterion 10.

The nine exported methods are the prop's authored-animation and door controls:
`run_anim_forward`, `run_anim_back`, `stop_anim`, `anim_time_get`, `anim_time_set`,
`play_bones_sound`, `stop_bones_sound`, `set_door_ignore_dynamics`,
`unset_door_ignore_dynamics`. Nothing about the physics body, the network path or the
hit callback is reachable from script; scripts open and close doors and scrub their
animation, and leave the solver alone.
