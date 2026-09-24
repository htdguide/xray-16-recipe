# src/xrGame/ai/monsters/burer/burer_script.cpp

> Exposes the burer class to the script layer under its frozen name.

**Needs** — [`burer.h`](burer.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

The per-creature script binding, identical in shape to every other creature's — see [`boar_script.cpp`](../boar/boar_script.cpp.md).

## `script_register`

**Contract** — Registers the type `CBurer` with the script machine, deriving from the game object facade, default-constructible. The forced-gravity override the class exposes is not bound here; scripts reach it through the shared creature surface.
