# src/xrGame/ai/monsters/pseudogigant/pseudogigant_script.cpp

> Registers the pseudogiant with the script layer as a named type deriving from the script-visible game object.

**Needs** — [`pseudo_gigant.h`](pseudo_gigant.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

Exposes the creature type to scripts under a fixed name, with the game-object facade as its
base and a default constructor. No methods are exported: scripts can recognise a pseudogiant
and use everything the game-object facade offers, but nothing giant-specific.

The file is separate from [`pseudo_gigant.cpp`](pseudo_gigant.cpp.md) only because the binding
layer's headers are expensive to include and the engine keeps every registration in its own
unit. A rebuild may merge them.

## `script_register`

**Contract** — called once at script-engine startup, from the chapter-wide registration sweep.
Declares the type, its base, and a constructor. The exported *name* is frozen: shipped scripts
address it by that string, so a rebuild may not rename it.

**Notes** — the exported constructor lets a script instantiate the type directly. Nothing
shipped does; creatures come from spawn records. It is present on every creature in the chapter
as a consequence of the binding layer's default shape rather than as a decision.
