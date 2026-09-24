# src/xrGame/StalkerOutfit.cpp

> Exports the stalker's suit to the script virtual machine.

**Needs** — [`StalkerOutfit.h`](StalkerOutfit.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

The stalker's suit class has no behaviour — like every other outfit, all of it is
configuration read by the generic outfit. The only reason the class has a source file at
all is that it must be declared to Lua, and the registration lives with the class rather
than in a central table.

## State

`Stateless.`

## `CStalkerOutfit::script_register`

**Contract** — registers the type `CStalkerOutfit`, deriving from the game object facade,
with a no-argument constructor and no methods. Runs once at script-engine bring-up. The
name is frozen by conformance criterion 10; scripts use it to identify and cast, nothing
more.
