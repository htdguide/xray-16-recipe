# src/editors/xrWeatherEngine/editor_environment_levels_manager.hpp

> Declares the level-to-weather assignment: which cycle each shipped level plays.

**Needs** — [`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md) · [`xrCore/Containers/AssociativeVector.hpp`](../../xrCore/Containers/AssociativeVector.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md)
**Tier floor** — T2: it holds two configuration files open for the editor's lifetime.

## Purpose

Declares the surface implemented in
[`editor_environment_levels_manager.cpp`](editor_environment_levels_manager.cpp.md). This
is the only sub-manager whose document is not part of the weather model at all — it edits
the level catalogue, because that is where a level's weather cycle is recorded.

## State

```text
RECORD LevelsManager
  levels          : map<text, (category : text, weather_cycle : text)>   # sorted by level name
  weathers        : WeathersManager     # the source of the cycle picker's options
  single_config   : Configuration       # the single-player level catalogue, held open
  mp_config       : Configuration       # the multiplayer level catalogue, held open
  property_holder : PropertyHolder
```

**Invariants** — both configurations stay open from load to teardown, because they are
rewritten on exit. The category is the text the grid groups rows under, and is one of two
literals.

## Exported units

- **`load`** — read both level catalogues and collect each level's cycle.
- **`fill`** — one grid row per level, grouped by category.
- **`save`** — declared; **never defined**. See the implementation.
