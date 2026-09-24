# src/xrGame/refreshable_obstacles_query_inline.h

> The refresh cadence: a two-metre query normally, a hundred-metre query on the tick — except that the tick test compares the wrong two quantities.

**Needs** — [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md) · [`xrEngine/device.h`](../xrEngine/device.h.md)
**Used by** — [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md)
**Tier floor** — T3: a clock comparison

## Purpose

Holds the only decision this type makes: which radius the next obstacles query runs at.

## State

```text
RECORD RefreshableObstaclesQuery extends ObstaclesQuery
  last_refresh_time : int    # milliseconds on the global clock; zero until the first
                             #   wide query
```

Two radii are fixed in code: **two metres** for the ordinary query and **one hundred metres**
for the periodic wide one, with a **one second** interval between wide queries. The ratio,
not the absolute values, is what carries: the wide query costs enough that it may run at
roughly the update rate of a single creature, and the narrow one cheaply enough to run every
time anyone asks.

## `refresh_radius`

**Intended contract** — answer with the large radius when at least the refresh interval has
passed since the last large query, stamping the time as it does so; answer with the small
radius otherwise.

```text
FUNCTION refresh_radius() -> real          # what this should decide
  IF now - last_refresh_time < refresh_interval THEN
    RETURN small_radius
  last_refresh_time = now
  RETURN large_radius
```

**What it actually does** — the elapsed-time test compares the global clock against the
*small radius plus the interval* rather than against the last refresh time plus the
interval. Since the small radius is two metres and the interval is one second, the threshold
is a constant 1002 milliseconds of uptime. The consequence is stark and worth stating
plainly: **for the first second after the process starts every query is narrow, and for the
entire rest of the session every query is wide.** The recorded timestamp is written but
never read.

**Notes**

- A rebuild should implement the intended contract, which is what the type's name, its
  constant and its field all describe. Reproducing the bug would mean every creature paying
  the hundred-metre obstacle scan on every query, which is not behaviour anybody chose.
- Be aware that fixing it changes navigation: creatures currently see distant obstacles
  immediately and would afterwards see them only once a second. If a rebuild finds paths
  routing through objects the original avoided, this is the first place to look.
- The radii are returned by reference to function-local constants, so callers hold a stable
  address. That is an artefact of the calling convention the base query uses and carries no
  decision.
