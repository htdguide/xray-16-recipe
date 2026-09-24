# src/xrGame/ai/monsters/snork/snork_script.cpp

> Registers the snork with the script layer as a named type deriving from the script-visible game object.

**Needs** — [`snork.h`](snork.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

Exposes the creature type to scripts under a fixed name, with the game-object facade as its
base and a default constructor. No methods are exported — notably not the creature's own
script-facing leap, which is therefore reachable only through code, not from a script holding
a snork.

## `script_register`

**Contract** — called once at script-engine startup from the chapter-wide registration sweep.
Declares the type, its base and a constructor. The exported name is frozen: shipped scripts
address it by that string.
