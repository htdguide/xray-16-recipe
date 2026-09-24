# src/xrGame/location_manager_inline.h

> Construction and the one accessor of a creature's terrain preference.

**Needs** — [`location_manager.h`](location_manager.h.md)
**Used by** — [`location_manager.h`](location_manager.h.md)
**Tier floor** — T3: a field read

## Purpose

Carries the terrain preference's constructor and its single accessor out of the
declaration, following the naming convention this part of the tree uses. A rebuild folds
them into the declaration.

## State

`Stateless.`

## Construction · `vertex_types`

**Contract** — construction binds the manager to the object whose preference it expresses
and starts with an **empty** mask list. Empty is meaningful: it means no preference has been
loaded, which the alife simulation reads as "any terrain is equally acceptable". A creature
that never loads a preference wanders freely.

`vertex_types` hands out the mask list for the alife simulation to score game graph vertices
against.
