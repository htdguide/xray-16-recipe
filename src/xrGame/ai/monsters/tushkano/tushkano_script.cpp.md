# src/xrGame/ai/monsters/tushkano/tushkano_script.cpp

> Exports the tushkano's class identity to the script layer, and nothing else.

**Needs** — [`tushkano.h`](tushkano.h.md) · [Seam: Script binding layer](../../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration

## Purpose

Registers the tushkano as a script-visible class deriving from the generic game object,
with a default constructor and no methods or properties of its own.

That is the whole file, and the emptiness is the point: scripts never call anything on a
tushkano directly. They talk to it through the generic game-object facade. The registration
exists so that a script can *name* the type — for a type test, or for the spawn machinery —
and so the binding layer knows where the tushkano sits in the exported inheritance chain.

The same file, verbatim apart from the name, exists for nearly every creature.

## `script_register`

**Contract** — declares the class, its script-visible name, its base class and a default
constructor into the script virtual machine. Called once during script-layer start-up. The
exported name is frozen by conformance criterion 10.
