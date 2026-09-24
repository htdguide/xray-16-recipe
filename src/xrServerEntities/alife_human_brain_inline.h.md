# src/xrServerEntities/alife_human_brain_inline.h

> The human brain's two field accessors.

**Needs** — [`alife_human_brain.h`](alife_human_brain.h.md)
**Used by** — [`alife_human_brain.h`](alife_human_brain.h.md)
**Tier floor** — T3.

## Purpose

Carries the accessors split out of [`alife_human_brain.h`](alife_human_brain.h.md): the
owning human record (re-declared at the narrower type, so callers do not have to convert
from the monster brain's), and the offline inventory handler.

## `owner` / `objects`

**Contract** — return the bound record and handler; both are always present.
