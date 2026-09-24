# src/xrGame/ai/monsters/cat/cat_script.cpp

> Exposes the cat class to the script layer under its frozen name.

**Needs** — [`cat.h`](cat.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

The per-creature script binding, identical in shape to every other creature's — see [`boar_script.cpp`](../boar/boar_script.cpp.md) for the reasoning that applies to all of them.

## `script_register`

**Contract** — Registers the type `CCat` with the script machine, deriving from the game object facade, default-constructible. No members are exported.
