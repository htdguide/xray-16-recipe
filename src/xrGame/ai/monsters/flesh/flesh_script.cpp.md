# src/xrGame/ai/monsters/flesh/flesh_script.cpp

> Exports the flesh to the script layer as a constructible class deriving from the script-visible
> game object.

**Needs** — [`flesh.h`](flesh.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

The flesh's script registration, identical in shape to every other creature's: one class name, the
game-object facade as its base, a default constructor, no creature-specific members. It is a
separate translation unit because the binding layer compiles with a different exception setting
from the rest of the game.

## `script_register`

**Contract** — register a script class named `CAI_Flesh` deriving from the script game-object
facade, with a default constructor. Called once at script-engine start-up.

**Notes** — the name is frozen by conformance criterion 10 and by the class identifier in the
shipped spawn data. Nothing flesh-specific is exposed: scripts reach a flesh through the generic
game-object and creature surfaces.
