# src/xrGame/script_entity_inline.h

> Reads the entity the action-queue mixin is half of.

**Needs** — [`script_entity.h`](script_entity.h.md)
**Used by** — [`script_entity.h`](script_entity.h.md)
**Tier floor** — T2: a field read

## Purpose

One accessor. It is separated from the header only because the entity type it returns is
forward-declared there; a rebuild merges it back.

## `object`

**Contract** — the entity this mixin is part of. Asserted non-empty: the mixin is only ever
constructed as part of an entity, so an empty value means construction was skipped, and
every caller would fault on the next line anyway.
