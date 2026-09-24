# src/xrEngine/PerformanceAlert.hpp

> Declares the one shipped filling of the performance-alert port.

**Needs** — [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md) · [`IPerformanceAlert.hpp`](IPerformanceAlert.hpp.md)
**Used by** — [`Device_Initialize.cpp`](Device_Initialize.cpp.md) · [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md) · [`Stats.cpp`](Stats.cpp.md) · [`xrSheduler.cpp`](xrSheduler.cpp.md) · [`xr_object_list.cpp`](xr_object_list.cpp.md)
**Tier floor** — T3: a colour, a size and a cursor

## Purpose

Declares the surface implemented in [`PerformanceAlert.cpp`](PerformanceAlert.cpp.md).

Exported units:

- `PerformanceAlert` — holds the alert colour (a fixed saturated red), the base font size
  the alerts are scaled from, the screen position alerts start at, and the running cursor.
  Constructed per frame by the statistics overlay.
- `Reset` — returns the cursor to the starting position; called once per frame before the
  dumps run.
- `Print` — appends one alert line. Implementation in the `.cpp`.

**Notes** — the alert colour is fixed in the constructor rather than configurable: an
alert that can be styled is an alert somebody will style into invisibility.
