# src/xrGame/ExplosiveScript.cpp

> Exports the explosive mixin to the script virtual machine with a single method: detonate now.

**Needs** — [`Explosive.h`](Explosive.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Declares the explosive capability to Lua under the name `explosive`, with no constructor
and no base class, because it is a mixin rather than an entity: script code never creates
one, it obtains one by casting a game object that happens to carry the capability. Classes
that inherit both the game object facade and this mixin — grenades, explosive items,
rockets — are registered elsewhere as deriving from both, and that is how a script reaches
the method.

## State

`Stateless.`

## `CExplosive::script_register`

**Contract** — registers the type `explosive` exposing one method, `explode`, which starts
the detonation sequence immediately (see [`Explosive.cpp`](Explosive.cpp.md): it sets the
explosion position and time and lets the normal explosive update run the effect and the
damage pass). The name and signature are frozen by
[conformance criterion 10](../../SYSTEM-REQUIREMENTS.md#6-conformance).

**Notes** — exporting a non-constructible mixin is the one thing here a rebuild must
reproduce deliberately: the binding layer has to allow a registered type that scripts can
receive and cast to but never instantiate.
