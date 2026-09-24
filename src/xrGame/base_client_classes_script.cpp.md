# src/xrGame/base_client_classes_script.cpp

> Exports the root of the client-object hierarchy to the script virtual machine, so that a script can derive its own game object.

**Needs** — [`base_client_classes_wrappers.h`](base_client_classes_wrappers.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_effector.h`](script_effector.h.md) · [`script_particles.h`](script_particles.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`Include/xrRender/RenderVisual.h`](../Include/xrRender/RenderVisual.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrEngine/Feel_Sound.h`](../xrEngine/Feel_Sound.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`base_client_classes_wrappers.h`](base_client_classes_wrappers.h.md)
**Tier floor** — T3: registration data

## Purpose

Every other script export in the game layer declares a *leaf*: a weapon, an artefact, a
zone. This one declares the trunk. It registers the abstract interfaces a client object is
assembled from — factory-constructible, scheduled, collidable, renderable — and then the
game object that combines them, in a form a script can **inherit from**.

That inheritance is the whole point and it is the most consequential thing in the file. A
mod can define a Lua class deriving from the game object and have the engine construct,
schedule, network-serialize and spawn it as a first-class entity. The methods listed below
are exactly the extension points where the engine will call *down* into script; everything
not listed cannot be overridden.

## State

`Stateless.`

## `CGameObject::script_register`

**Contract** — registers into the script virtual machine, once at script-engine bring-up.

- **`CBlend`** — an opaque handle for a playing animation. No members; scripts pass it back.
- **`IKinematicsAnimated`** — one method, play a named animation cycle. The return is
  discarded, which is why the export wraps the real call rather than binding it directly.
- **`IRender_Visual`** — one method, ask a visual for its animated-skeleton facet. This plus
  the previous entry is the entire animation surface a script has on an arbitrary object.
- **`DLL_Pure`** — the factory-object interface, constructible and with an overridable
  construct step. **The name is historic and deliberately preserved**: shipped mod scripts
  name it, and renaming it to match the engine's own type name would break them. This is a
  frozen name that does not correspond to any internal name, and a rebuild must keep it.
- **`ISheduled`** — the scheduler interface, overridable so a script class can supply its own
  update rate and update body.
- **`IRenderable`** and **`ICollidable`** — registered as types so the hierarchy below them
  resolves. Neither exposes a method.
- **`CGameObject`**, deriving from all four, constructible, with:
  - **`_construct`** — the construction step.
  - **`net_Spawn`** — build the client object from a server record. Overridable, and
    returning false from it aborts the spawn.
  - **`net_Import`** / **`net_Export`** — read and write the object's network update.
    Overridable, which means a script-defined entity can define its own wire format.
  - **`use`** — the interaction verb: another object has used this one.
  - **`Visual`**, **`getVisible`**, **`getEnabled`** — readers.

Names and signatures are frozen by conformance criterion 10.

**Invariants** — each overridable method is registered twice: once as the virtual entry the
script overrides, and once as a static that invokes the *engine's* implementation. That
pairing is what lets a script override `net_Spawn`, do its own work, and then call the base
behaviour. Without it an override would have to reimplement the entire spawn, which no
script can. A rebuild's binding layer must offer the same "call the base implementation"
escape or every one of these overrides becomes useless.

**Notes** — two unrelated headers are included purely to force the linker to keep symbols
that nothing in this translation unit references — the script-driven camera effector and
particle helpers, whose registrations would otherwise be dropped from the final binary. That
is an artefact of static-library linking with no equivalent obligation in a rebuild; what
survives is the fact that *those two registrations exist and must reach the script machine*.
