# src/utils/mp_balancer/statistics_collector.hpp

> Declares the pass that flattens the multiplayer item set into spreadsheet tables, one table per item family.

**Needs** — [`pch.h`](pch.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md)

**Used by** — [`entry_point.cpp`](entry_point.cpp.md) · [`statistics_collector.cpp`](statistics_collector.cpp.md)

**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`statistics_collector.cpp`](statistics_collector.cpp.md). It borrows the loaded
configuration from [`wpn_collection.cpp`](wpn_collection.cpp.md) rather than loading its
own, because the expensive part — parsing a whole game's configuration — has already been
paid for by the time this runs.

## Exported units

- `statistics_collector` — one export run, constructed over an already-loaded weapon
  collection.
- `load_settings` — read the table definitions from the same job description the merge
  pass uses.
- `save_files` — write every table.
- `save_file` — write one table.
- `get_most_acceptable_group` — decide which table an item belongs in.
