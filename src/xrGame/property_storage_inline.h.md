# src/xrGame/property_storage_inline.h

> Implementations of the planner's answer board.

**Needs** — [`property_storage.h`](property_storage.h.md)
**Used by** — [`property_storage.h`](property_storage.h.md)
**Tier floor** — T3: linear search over a small list

## Purpose

Supplies the three bodies declared in [`property_storage.h`](property_storage.h.md), where
the contracts live. Separate only because the calls are hot enough that the original wanted
them inlined at every call site; a rebuild merges the two files.

The only content worth carrying: both read and write are a **linear scan comparing the
question identifier**, and the read fails hard rather than defaulting when the question is
absent.
