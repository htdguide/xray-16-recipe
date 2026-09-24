# src/xrGame/base_client_classes_wrappers.h

> The shim types that let a script class inherit from an engine client class and have the engine call back into it.

**Needs** — [`base_client_classes_script.cpp`](base_client_classes_script.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`Entity.h`](Entity.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrServerEntities/xrServer_Object_Base.h`](../xrServerEntities/xrServer_Object_Base.h.md) · [`xrEngine/ISheduled.h`](../xrEngine/ISheduled.h.md) · [`xrEngine/IRenderable.h`](../xrEngine/IRenderable.h.md) · [`xrEngine/ICollidable.h`](../xrEngine/ICollidable.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [`xrEngine/EngineAPI.h`](../xrEngine/EngineAPI.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`base_client_classes_script.cpp`](base_client_classes_script.cpp.md) · [`fs_registrator_script.cpp`](fs_registrator_script.cpp.md)
**Tier floor** — T1: it exists to bridge two object models, and the shape it takes is dictated by both

## Purpose

A script class that derives from an engine class creates a two-way problem. The script side
must be able to call the engine's methods, which is ordinary binding. But the *engine* must
be able to call the script's overrides — the scheduler must tick a script-defined object,
the network layer must ask it to serialize itself — and the engine only knows how to call
its own virtual methods on its own types.

This header supplies the missing piece: for each extension point, a type that implements the
engine's method by dispatching into the script instance, paired with a static that invokes
the engine's own implementation so an override can still reach the base behaviour. Nothing
here is game logic. What a rebuild owes is the *capability* — a script subclass whose
overrides the engine invokes, and which can call up to its base — not this arrangement of
types.

## State

`Stateless.` Every type here is a dispatch shim over an engine object.

## the extension points

**Contract** — this is the authoritative list of what a script-defined client object may
override, and it is short because each entry costs a shim:

```text
INTERFACE script-overridable client object
  FUNCTION construct() -> object      # the factory step
  FUNCTION spawn(server_record) -> bool   # build from the authoritative record; false aborts
  FUNCTION import_network_update(packet)
  FUNCTION export_network_update(packet)
  FUNCTION use(who) -> bool           # the interaction verb
  FUNCTION scheduled_update(dt)       # from the scheduler
  FUNCTION schedule_scale() -> real   # the scheduler's cost hint; defaults to 1
```

**Invariants** — every overridable point has a companion "call the engine's version"
operation. An override that does not call it replaces the engine behaviour entirely, which
for the spawn path means the object is never registered anywhere and is a script bug rather
than an engine one. A rebuild must expose both halves.

## creature hit callbacks

**Contract** — a script class deriving from the *creature* level of the hierarchy may also
override how it reacts to a hit: the signal (a hit of some strength landed on some bone,
from someone) and the impulse (the physical push, in world and local directions).

**Invariants** — these two have no engine implementation to fall back on; they are pure at
that level. Their "call the base" companions therefore do not call anything — they log a
script error saying a pure method was invoked. That is the only honest thing they can do,
and it means a script class at this level that does not override both is broken, discovered
at the moment of the first hit rather than at registration.

## the deferred registration group

**Contract** — several script-visible utility types are declared here as empty shells
carrying only a registration hook: the flag set, the colour, the vector, the matrix, the
filesystem accessor, the configuration-file reader and the network packet. Two of them
declare a *dependency* on another registration — the reader on the vector, the packet on
the vector and the matrix — which is the mechanism by which the script engine orders
registration so that a type is never exported before the types appearing in its signatures.

**Notes** — the ordering requirement is real and easy to miss in a rebuild: exporting a
method taking a vector before the vector type exists leaves the binding unusable in a way
that only shows up when a script calls it.

**Notes** — the factory shim also stubs out the class-identifier accessor with a fixed
invalid value. A script-defined class has no entry in the shipped class-identifier table —
it is created by script, not by a spawn record — so it has no identifier to report, and the
invalid value is how the rest of the engine learns that.
