# src/xrServerEntities/script_value_inline.h

> The trivial bodies of the shadow cell's constructor and name accessor.

**Needs** — [`script_value.h`](script_value.h.md)
**Used by** — [`script_value.h`](script_value.h.md)
**Tier floor** — T3: two field assignments.

## Purpose

Carries the inline definitions split out of [`script_value.h`](script_value.h.md) so that
the header stays a declaration. There are no decisions here — the constructor stores the
table reference and the name, and the accessor returns the name. In a rebuild this file does
not exist.
