# src/xrGame/movement_manager_impl.h

> An empty file: the template-body header the movement manager never needed.

**Needs** — _(none)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: nothing

## Purpose

The file contains no declarations. Its siblings across this directory follow a convention
of three headers per component — the class, its inline accessors, its template bodies —
and this is the third one, left empty because `CMovementManager`'s only template member is
a one-line forward that lives in [`movement_manager_inline.h`](movement_manager_inline.h.md)
instead. A rebuild deletes it.

## State

`Stateless.`
