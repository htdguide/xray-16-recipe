# src/xrGame/ai/monsters/boar/boar_script.cpp

> Exposes the boar class to the script layer as a named type derived from the script-visible game object.

**Needs** — [`boar.h`](boar.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one binding declaration

## Purpose

Every creature class gets one of these files and they are all the same shape: declare the class to the script virtual machine under a frozen name, state its base as the script-visible game object facade, and give it a default constructor. The name is part of the modding surface and cannot change.

It is a separate translation unit only because the binding layer's headers are expensive to include; a rebuild whose binding layer is cheap should fold it into the class file.

## `script_register`

**Contract** — Registers the type `CAI_Boar` with the script machine, deriving from the game object facade, constructible with no arguments. Called once during script-machine setup. No other members are exported; scripts reach the boar's behaviour through the shared creature surface.
