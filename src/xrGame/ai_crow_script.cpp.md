# src/xrGame/ai_crow_script.cpp

> Exports the crow to the script layer.

**Needs** — [`ai/crow/ai_crow.h`](ai/crow/ai_crow.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration.

## Purpose

Each game class declares its own script surface next to itself rather than in a central
table; this is the crow's. It exists as a separate file so the crow's implementation need
not be compiled against the binding layer.

## `script_register`

**Contract** — Registers the crow as a script class named `CAI_Crow`, derived from the
game object facade, with a default constructor and no methods or properties of its own.
The crow is scriptable only so that scripts can name its type — spawn it, test an object
against it — not so they can steer it.

**Notes** — The exact exported name is frozen by conformance criterion 10; shipped scripts
refer to it.
