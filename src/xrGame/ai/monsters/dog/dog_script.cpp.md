# src/xrGame/ai/monsters/dog/dog_script.cpp

> Exports the dog to the script layer as a constructible class deriving from the script-visible
> game object.

**Needs** — [`dog.h`](dog.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

Every creature class is registered into the script virtual machine under its own name, with the
game-object facade as its base and a no-argument constructor. This file is the dog's registration
and it is exactly that and nothing more.

It is a separate translation unit from [`dog.cpp`](dog.cpp.md) for a build reason that survives
as a real constraint: the binding layer is compiled with a different exception setting from the
rest of the game, so registration code cannot share a compilation unit with behaviour. A rebuild
whose binding layer has no such split can fold this file into the creature's own.

## `script_register`

**Contract** — register a script class named `CAI_Dog`, deriving from the script game-object
facade, with a default constructor and no additional methods or properties. Called once during
script-engine start-up, before any shipped script runs.

**Notes** — the class exposes **no dog-specific methods at all**. What the shipped scripts do with
a dog, they do through the generic game-object and creature surfaces; the registration exists so
that a script can name the type — for spawn-class dispatch and for type tests — not so it can
call into the dog.

The name is frozen by conformance criterion 10: shipped scripts refer to it verbatim, and the
class identifier in the level's spawn data resolves to this registration.
