# src/xrGame/alife_human_brain_script.cpp

> Exports the offline human brain to the script layer as a named type with no members.

**Needs** — [`alife_human_brain.h`](../xrServerEntities/alife_human_brain.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one registration.

## Purpose

Registers the human brain as a script class derived from the monster brain. No constructor,
no methods, no properties — scripts can *hold* one and can use it wherever a monster brain
is expected, but cannot call anything on it directly.

**Notes** — The registration exists so that the inheritance is visible to the binding layer:
a script receiving a human's brain through the monster-brain interface must be able to
recognize it. The exported name is frozen by conformance criterion 10.
