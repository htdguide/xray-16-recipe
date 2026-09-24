# src/xrGame/script_effector.cpp

> A post-processing effector whose per-frame parameter evaluation is written in script.

**Needs** — [`script_effector.h`](script_effector.h.md) · [`Actor.h`](Actor.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_effector.h`](script_effector.h.md)
**Tier floor** — T2: it decides post-processing parameters; the parameters themselves are consumed by the renderer

## Purpose

The actor's camera owns a stack of *post-process effectors*, each of which gets a chance
every frame to modify a shared block of screen parameters — blur, desaturation, film
grain, colour tint, double vision. This file lets one of those effectors be authored in
script, so that a mod can darken the screen during a cutscene or pulse it during a
psi-attack without touching the engine.

## State

```text
RECORD ScriptEffector
  effector_type : int     # the effector's slot identity; also its key for removal
  # the rest of an effector's state (lifetime, elapsed time) lives in the base
```

**Invariant** — `effector_type` duplicates a value the base also stores. It is kept here
because removal is by *type*, not by instance: adding a second effector of an existing
type replaces the first, and `remove` needs the key after the base may already have
forgotten it.

## `construct(type, duration)`

**Contract** — creates an effector of the given slot type that expires after `duration`
seconds. The effector does *not* loop; when its time runs out the camera drops it.

## `process(parameters) -> bool`

**Contract** — called once per frame while the effector is alive, with the shared
parameter block. Returns whether the effector wants to keep living: false retires it early.
Script overrides this; the base implementation just advances the effector's own clock and
reports whether time is left.

**Notes**

Two entry points exist, differing only in whether the parameter block is passed by
reference or by handle. The handle form is the one the script layer can bind, because the
binding layer cannot express the reference form's ownership. A rebuild needs only one.

## `add`

**Contract** — pushes this effector onto the actor's camera effector stack. Ownership
transfers: from here on the camera decides when the effector dies, and the script must not
free it. There is exactly one actor, so no target is passed.

## `remove`

**Contract** — removes whatever effector currently occupies this effector's *type* slot on
the actor's camera and destroys it. Calling it on an effector that was never added, or was
already retired, is harmless.

**Notes**

Removal by type rather than by identity is what makes `start`/`finish` usable from script
without the script holding the object alive: a script can call `finish` on a fresh
effector value of the same type and still cancel the running one.
