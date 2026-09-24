# src/xrGame/ui/FractionState_inline.h

> The field accessors of the older faction record.

**Needs** — [`FractionState.h`](FractionState.h.md)
**Used by** — [`FractionState.h`](FractionState.h.md)
**Tier floor** — T3: field accessors

## Purpose

Pure accessors over the record declared in [`FractionState.h`](FractionState.h.md). Each reads
or writes one field; none decides anything. They exist as a separate file for the same reason
as [`FactionState_inline.h`](FactionState_inline.h.md): the script binding needs a callable
pair per exported property.

Unlike its sibling there are no indexed slots here, so there is no name-versus-index
duplication — the file is exactly one accessor pair per field and holds no decision at all.
