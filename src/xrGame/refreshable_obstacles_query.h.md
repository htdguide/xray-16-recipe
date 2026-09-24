# src/xrGame/refreshable_obstacles_query.h

> Declares an obstacles query that periodically widens its own search radius so distant obstacle changes are eventually noticed.

**Needs** — [`obstacles_query.h`](obstacles_query.h.md) · [`refreshable_obstacles_query_inline.h`](refreshable_obstacles_query_inline.h.md)
**Used by** — [`refreshable_obstacles_query_inline.h`](refreshable_obstacles_query_inline.h.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md) · [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md)
**Tier floor** — T3: one timestamp and a radius choice

## Purpose

An obstacles query asks which nearby objects block navigation. Asking over a wide radius is
expensive and asking over a narrow one misses an obstacle that appeared out of range and has
since become relevant. This type is the compromise: it answers with a small radius most of
the time and a large one on a periodic tick, so the narrow query stays cheap and the wide
one eventually catches up.

The substance is in [`refreshable_obstacles_query_inline.h`](refreshable_obstacles_query_inline.h.md).

Exported units:

- `refreshable_obstacles_query` — an obstacles query with a refresh cadence.
- `refresh_radius` — the radius to use for *this* query, small or large by the cadence.

**Notes** — the refresh interval is a fixed one second, written in this file as a constant
rather than read from configuration. Nothing else in the engine references it, and no
comment justifies the value; treat it as a tuning knob a rebuild may expose.
