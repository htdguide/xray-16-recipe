# src/xrGame/property_storage_script.cpp

> Exports the planner's answer board to the script layer.

**Needs** — [`property_storage.h`](property_storage.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration only

## Purpose

Lua-authored evaluators and operators are handed a storage and must be able to read and
write it. This file names that surface.

## `script_register`

**Contract** — registers `property_storage` with a default constructor, `set_property`
(question, answer) and `property` (question) → answer. `clear` is deliberately **not**
exported: clearing the board is the planner's business, and a script that cleared it would
strand every operator mid-plan.

**Invariants** — reading an unanswered question raises on the script side too, which is how
a mod author learns the evaluator they expected is not installed.
