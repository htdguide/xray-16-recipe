# src/xrGame/ai/monsters/zombie/zombie_script.cpp

> Exports the zombie's class identity to the script layer.

**Needs** — [`zombie.h`](zombie.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration

## Purpose

Registers the zombie as a script-visible class deriving from the generic game object, with a
default constructor and nothing else. Scripts reach a zombie's behaviour through the generic
game-object facade; the registration exists so the type can be named.

Notably, **none of the zombie's own mechanics are exposed**: a script cannot make a zombie
feign death, cannot read how many feigns it has left, and cannot query whether one is in
progress. The feigned-death mechanic is entirely engine-side.

## `script_register`

**Contract** — declares the class, its script-visible name, its base class and a default
constructor into the script virtual machine. Called once at script-layer start-up. The
exported name is frozen by conformance criterion 10.
