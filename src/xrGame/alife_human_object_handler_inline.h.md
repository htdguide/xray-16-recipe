# src/xrGame/alife_human_object_handler_inline.h

> The offline inventory manager's binding to its record.

**Needs** — [`alife_human_object_handler.h`](alife_human_object_handler.h.md)
**Used by** — [`alife_human_object_handler.cpp`](alife_human_object_handler.cpp.md) · [`alife_human_object_handler.h`](alife_human_object_handler.h.md)
**Tier floor** — T3: field access.

## Purpose

Construction stores the human server record the handler acts on, asserting it is present;
the accessor hands it back with the same assertion. The handler owns nothing and outlives
nothing — it is a behaviour attached to one record. A rebuild folds both into the type.
