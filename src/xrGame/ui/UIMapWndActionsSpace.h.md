# src/xrGame/ui/UIMapWndActionsSpace.h

> The map view's planning vocabulary: five world properties and four operators, shared between
> the planner and the actions that satisfy it.

**Needs** — _(none)_
**Used by** — [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md) · [`UIMapWndActions.h`](UIMapWndActions.h.md)
**Tier floor** — T3: two enumerations.

## Purpose

A constants header, and therefore the substance holder for the map planner's *vocabulary*. The
planner in [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md) searches over these, so the names
here are the state space a rebuild must reproduce — though the values themselves are internal
and reach no file or wire.

## `EWorldProperties`

```text
ENUM ViewProperty
  target_map_shown     # the goal place is within reach of the visible area
  map_minimized        # the world map is at its minimum zoom
  map_resized          # the current animation has finished resizing
  map_idle             # the view has settled; the planner's goal
  map_centered         # declared, never used
  dummy
```

## `EWorldOperators`

```text
ENUM ViewOperator
  resize      # animate toward the target at the current zoom
  minimize    # animate out to the minimum zoom
  idle        # settle, and latch the settled state
  center      # declared, never registered
  dummy
```

**Notes** — `map_centered` and `center` are declared and never used; the centring behaviour
ended up inside the resize operator. A rebuild drops both. Two further properties the planner
uses are *not* in this enumeration at all — it addresses them by bare index, which is noted in
the planner's twin.
