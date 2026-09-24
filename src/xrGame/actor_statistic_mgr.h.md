# src/xrGame/actor_statistic_mgr.h

> Declares the player's scorecard manager, implemented in [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md).

**Needs** — [`actor_statistic_defs.h`](actor_statistic_defs.h.md)
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md) · [`level_script.cpp`](level_script.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the manager that owns the player's persistent scorecard registry and the
find-or-create access to it. Substance is in
[`actor_statistic_mgr.cpp`](actor_statistic_mgr.cpp.md).

Exported units:

- `GetSection` — the named section, created on first use.
- `AddPoints` in two forms — add a count and a per-unit score, or set a displayed text
  value.
- `GetSectionPoints` — a section's total, or the aggregate when asked for the reserved
  name `total`; a negative one means *not scorable*.
- `GetCStorage` — a read-only view of every section, for the statistics screen.

**Notes** — the writable view of the storage is private and the read-only view is public,
which is the right split: everything that changes the scorecard should go through the
adders so that sections and tallies are created consistently.
