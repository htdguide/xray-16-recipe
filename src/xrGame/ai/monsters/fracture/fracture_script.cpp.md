# src/xrGame/ai/monsters/fracture/fracture_script.cpp

> Exports the fracture to the script layer as a constructible class deriving from the
> script-visible game object.

**Needs** — [`fracture.h`](fracture.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Script virtual machine](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

The fracture's script registration, identical in shape to every other creature's. A separate
translation unit because the binding layer compiles with a different exception setting from the
rest of the game.

## `script_register`

**Contract** — register a script class named `CFracture` deriving from the script game-object
facade, with a default constructor and no creature-specific members. Called once at script-engine
start-up. The name is frozen by conformance criterion 10 and by the shipped spawn data's class
identifier.
