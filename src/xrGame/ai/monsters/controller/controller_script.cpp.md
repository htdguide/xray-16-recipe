# src/xrGame/ai/monsters/controller/controller_script.cpp

> Exports the controller creature to the script layer as a bare class name with a default constructor and nothing else.

**Needs** — [`controller.h`](controller.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding registration

## Purpose

Every game class that scripts may name has a registration like this one. The controller's is
as small as a registration gets: the class name and a default constructor, deriving from the
generic game-object facade.

The registration is in its own file for a build reason — the binding layer's headers are
heavy and are kept out of the creature's own translation unit — and that reason does not
survive a rebuild. What survives is the *fact* of the export and its exact shape, because
conformance criterion 10 freezes the script surface.

## State

Stateless.

## `script_register`

**Contract** — register the class under the name `CController`, deriving from the game-object
facade, with a default constructor. No method, property or enumeration is exported.

**Notes** — the shipped scripts therefore cannot ask a controller anything about its thralls,
its mental state or its set-piece attack; they can only recognize one by type and use the
generic game-object surface. A rebuild must export exactly this and no more, because a
*wider* surface is as much a compatibility change as a narrower one for scripts that
enumerate members.

The default constructor is exported even though scripts never construct creatures — spawning
goes through the class-identifier factory. It is present because the binding layer's class
declaration wants one.
