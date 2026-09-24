# src/xrGame/physic_item_inline.h

> An empty file: the inline-accessor header the physical item never needed.

**Needs** — _(none)_
**Used by** — [`physic_item.h`](physic_item.h.md)
**Tier floor** — T4: nothing

## Purpose

The file contains no declarations. Its siblings in this directory follow a convention of a
class header plus an inline header, and this is the inline half, left empty because
`CPhysicItem` has no accessors worth inlining — its one field is public. A rebuild deletes it.

## State

`Stateless.`
