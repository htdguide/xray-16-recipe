# src/xrGame/alife_online_offline_group_brain_inline.h

> The two accessors of the squad brain.

**Needs** — [`alife_online_offline_group_brain.h`](alife_online_offline_group_brain.h.md)
**Used by** — [`alife_online_offline_group_brain.h`](alife_online_offline_group_brain.h.md)
**Tier floor** — T3: field reads

## Purpose

Bodies for the always-inlined accessors of `CALifeOnlineOfflineGroupBrain`, split out of
the header as a C++ habit. A rebuild folds them into the type. Substance is in
[`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md).

## Accessors

**Contract** — `object` returns the squad this brain drives; `movement` returns the
owned movement manager. Both are required to be present for the brain's whole lifetime —
they are references, not optional links, and a rebuild should say so in the type rather
than assert it at each use.
