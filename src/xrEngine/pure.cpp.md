# src/xrEngine/pure.cpp

> The compilation anchor for the signal registry; the substance is in the header.

**Needs** — [`pure.h`](pure.h.md)
**Used by** — reached through its declarations in [`pure.h`](pure.h.md); callers name that, not this file.
**Tier floor** — T4: no behaviour.

## Purpose

Empty. The signal set and the registry are entirely generic over the signal type, so all
of it lives in [`pure.h`](pure.h.md); this file exists only to give the module a
translation unit. A rebuild has nothing here.

## State

Stateless.
