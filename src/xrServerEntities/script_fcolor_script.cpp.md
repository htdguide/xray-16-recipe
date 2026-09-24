# src/xrServerEntities/script_fcolor_script.cpp

> Exports the four-channel floating-point colour to scripts.

**Needs** — [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Publishes the colour type under the script name `fcolor`: the four channels as readable and
writable fields, and three ways to set it — from four components, from another colour, and
from a packed 32-bit value. The packed form is what makes it useful, since configuration and
the user interface both carry colours as packed integers while the engine works in floats.

**Invariants** — every set form returns the colour itself so calls chain, with the binding
declaring the returned reference to be the first argument.

**Notes** — the channel order in the packed conversion is the engine's, and a script that
composes a packed value by hand has to match it. Nothing here documents which order that is;
it belongs to the colour type in the core layer.
