# src/xrGame/object_handler_inline.h

> Three accessors on the object handler.

**Needs** — [`object_handler.h`](object_handler.h.md)
**Used by** — [`object_handler.h`](object_handler.h.md)
**Tier floor** — T3: field access

## Purpose

Inline accessors split out because the original language wants inline bodies after the class
body. The split is arbitrary; a rebuild folds them into the type.

## State

`Stateless.`

## Accessors

**Contract** — `hammer_is_clutched` reports whether the creature's weapon hammer is held back
(a per-creature animation state the shot effector reads). `infinite_ammo` reports the flag
taken from the server record at spawn. `planner` hands out the object-handling planner,
asserting it exists — a construction invariant, so checked builds only.
