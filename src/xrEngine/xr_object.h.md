# src/xrEngine/xr_object.h

> The contract every client object on a level satisfies — the widest interface in the engine, and the seam across which the engine drives the game module without knowing what a weapon is.

**Needs** — [`ISheduled.h`](ISheduled.h.md) · [`IRenderable.h`](IRenderable.h.md) · [`ICollidable.h`](ICollidable.h.md) · [`EngineAPI.h`](EngineAPI.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md) · [`xrCommon/misc_math_types.h`](../xrCommon/misc_math_types.h.md) · [`xrGame/game_object_space.h`](../xrGame/game_object_space.h.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrServerEntities/xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md)
**Used by** — [`WallmarksEngine.cpp`](../Layers/xrRender/WallmarksEngine.cpp.md) · [`dx113DFluidObstacles.cpp`](../Layers/xrRenderDX11/3DFluid/dx113DFluidObstacles.cpp.md) · [`CameraBase.h`](CameraBase.h.md) · [`Feel_Touch.cpp`](Feel_Touch.cpp.md) · [`Feel_Vision.cpp`](Feel_Vision.cpp.md) · [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) · [`IGame_ObjectPool.h`](IGame_ObjectPool.h.md) · [`ISheduled.cpp`](ISheduled.cpp.md) · [`ObjectDump.cpp`](ObjectDump.cpp.md) · [`ObjectDump.h`](ObjectDump.h.md) · [`cf_dynamic_mesh.cpp`](cf_dynamic_mesh.cpp.md) · [`xr_collide_form.cpp`](xr_collide_form.cpp.md) · [`xr_efflensflare.cpp`](xr_efflensflare.cpp.md) · [`xr_object_list.cpp`](xr_object_list.cpp.md) · _and 9 more_
**Tier floor** — T1 as written, because the property word is a packed bitfield whose layout the debug tooling reads as a unit. As a *contract* it is T2: nothing about the obligations requires manual memory.

## Purpose

The engine owns the frame, the registry, the renderer and the scheduler; the game module
owns what things *are*. This file is the entire vocabulary the first uses to drive the
second. Every client object — the actor, a rifle, a mutant, an anomaly, a door — satisfies
it, and the engine never learns anything more specific.

It is deliberately enormous, and the size is the honest record of a decision: rather than
many narrow interfaces negotiated per-subsystem, there is one interface that composes five
role interfaces and then adds everything else. That has one consequence a rebuilder must
plan for — **the whole game layer is behind one type**, so the engine/game split is a
compile-time boundary, not a runtime one, and the interface changes whenever the game
gains a concept.

## Composition

A client object is simultaneously five things, each with its own contract declared
elsewhere:

| Role | Obligation |
|---|---|
| factory product | has a class id, can construct a sibling of its own type |
| [spatial](../xrCDB/ISpatial.h.md) | occupies a place in the visibility/query tree; can register, move and unregister itself there |
| [scheduled](ISheduled.h.md) | can be advanced by a time delta on the scheduler's budget, and can say whether it currently needs to be |
| [renderable](IRenderable.h.md) | contributes a visual and per-object render state |
| [collidable](ICollidable.h.md) | owns a collision form |

The single interface then adds the obligations below. Any rebuild that wants a narrower
seam should cut along these groups, because they are already the natural joints.

## State an implementor must carry

```text
RECORD ObjectProperties          # one packed word; the debug overlay reads it whole
  net_id           : int (16-bit)
  activation_count : int (8-bit) # nested enable/disable requests; see below
  enabled          : bool
  visible          : bool
  destroying       : bool
  net_local        : bool        # this process is authoritative for the object
  net_ready        : bool        # spawn has completed
  net_server_update: bool        # the server wants this object's state
  is_crow          : bool        # queued for this frame's update pass
  pre_destroy      : bool

RECORD SavedPosition             # ring of recent transforms, for interpolation and rewind
  time     : int
  position : (real, real, real)
```

The counted activation is the load-bearing detail in that word. "Wants per-frame updates"
is not a boolean owned by one caller — a weapon being fired, an animation playing and a
script all independently want the object awake. Each asks and each releases; the object
sleeps when the count reaches zero. A rebuild that uses a plain flag will get objects that
freeze when any one of several owners finishes.

## The obligation groups

### Identity and lifetime

Class id; three names (the instance name, the configuration section it was built from, and
the visual model's name); a network id with the reserved *none* value; the spawn record's
private configuration block. Spawn takes the authoritative server record and returns
whether the object accepted it — a refusal is how a client declines to materialise
something it cannot represent.

### The update trio

- **pre-update** — runs on every object in this frame's workload before any of them update.
- **update** — the per-frame advance. Takes no time delta: the frame's delta is global, and
  making it a parameter would invite objects to integrate with a delta other than the
  frame's.
- **post-update** — runs on *every* object, in or out of the workload, and is told which.
  This is where work that must not be skipped lives.

Alongside them, two frame stamps the registry owns: the last frame this object updated, and
the frame it last asked to be in the workload. Both exist so that "once per frame" and
"requested once" are decidable without searching lists.

### Parentage

An object may be attached to another (carried, mounted, holstered). The interface exposes
the immediate parent, the root of the chain, and a setter with a flag for "this is
happening because the parent is being destroyed", which suppresses the normal detach
behaviour. Four hooks fire around a change — before and after becoming a child, before and
after becoming independent — because the object needs to read its old transform before the
change and its new one after.

**Invariant**: the parent chain is acyclic and the registry relies on it terminating.

### Transform and geometry

World transform as a matrix, with position and direction as views onto it; a centre, a
radius and a bounding box; the visibility sector the object currently sits in; a forced
transform that moves the object without the usual notifications (teleport, network
correction); and a hook fired when the matrix changes so derived state can be invalidated.

A short history of recent positions is kept and readable by index — the substrate for
interpolation and for the network's correction-prediction.

### Reference dropping

One method, called on every surviving object for every object about to be destroyed. The
implementor's obligation is absolute: after it returns, it holds no reference to the named
object. Failing this is the most common class of crash in the engine's history, which is
why the notification is broadcast to everyone rather than to registered listeners only.

### Network

Export and import of per-frame state; save and load of persistent state; spawn and destroy;
input import; a relevance query that decides whether the object is worth a packet at all;
migration between authorities; and the correction-prediction bracket — four hooks around
the physics step (before, between correction and prediction, after, plus an activation step
counter) that let an object reconcile a server correction against locally simulated motion
without the physics layer knowing anything about networking.

### The cast bank

Roughly twenty methods of the form "am I an inventory owner / an actor / a weapon / a
restrictor", each returning the object as that type or nothing. This is a dynamic type test
written out by hand, and the comment in the source says why: it replaced a checked
downcast that was hot enough to matter. In a rebuild this is a smell, not a design — the
right fix is capability interfaces or a component lookup, and the list above is a faithful
inventory of which capabilities the engine actually asks about.

### Script and game-facing surface

The Lua facade object; the callback table keyed by event kind; the script binder; the
usable-object protocol (can this be used, by whom, what tip text does it show); the AI
evaluation-function type codes (creature, equipment, weapon, anomaly, detector) used by the
alife's coarse reasoning; the story id that ties an object to authored plot; obstacle and
ai-location accessors.

### Animation movement control

An object may hand control of its own transform to an animation for a while — root motion.
The interface is create / destroy / update / query, plus a read-only accessor to the
controller. The creation takes the blend being played, an optional starting transform and
whether the motion is object-local or world-absolute.

### Debug-only obligations

An extra update-frame stamp used to assert that overridden update methods called their
base; a property-word dump; a skeleton draw; and an opt-in per-object render-time hook.
These exist to catch the two silent failures the design allows — an object that never
updates, and an object that updates twice.

## Notes

`CROW_RADIUS` (30) and its square (60) are defined here and are the distance inside which
an object makes itself part of the frame's workload. The squared constant is wrong as a
square — 30 squared is 900 — so it is a second, larger radius rather than a derived value,
and the two are used for different tests by the game layer. **This is one of the places the
recipe cannot recover intent**: the naming asserts a relationship the values do not have.

The interface names a class from the *game* module and two from the *server entities*
module. That is the `xrEngine` ⇄ `xrGame` cycle named in the preface, and this file is
where it is widest. A rebuild should invert it: the engine declares what it needs of an
object, and the game supplies an adapter, rather than the engine's own header reaching into
the game's vocabulary.
