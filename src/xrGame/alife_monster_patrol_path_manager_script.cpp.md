# src/xrGame/alife_monster_patrol_path_manager_script.cpp

> Exports the offline patrol cursor to the script layer.

**Needs** — [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md) · [`xrAICore/Navigation/PatrolPath/patrol_path.h`](../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a registration table

## Purpose

Binding only. Publishes `CALifeMonsterPatrolPathManager` under that name. This is the
surface a smart terrain's script uses to put an offline creature on a named patrol and
tune how it walks it, and it is the busiest of the three offline-movement bindings
because every traversal setting is script-authored in the shipped game logic.

## Exported surface

**Contract** — no constructor. Methods:

- `path` — by name only. The reference-taking overload is *not* exported: a script names
  a path, it does not hold one, so a script cannot install a path object the level does
  not own.
- `start_type`, `route_type` — overloaded getter and setter over the two traversal
  enumerations.
- `use_randomness` — overloaded getter and setter.
- `start_vertex_index` — setter only; there is no reason to read back a value the script
  itself supplied.
- `actual`, `completed` — cursor status.
- `target_game_vertex_id`, `target_level_vertex_id`, `target_position` — the current
  destination. The position is exported through an adapter that yields a *copy* rather
  than the internal reference, because a script holding a live reference into shared,
  immutable level data could otherwise observe it change when the path is swapped.

**Notes** — the two enumerations must be registered separately (with the patrol path data
model) for these setters to be callable with named constants; a rebuild that exports the
methods but forgets the enumeration values leaves the surface technically complete and
practically unusable from the shipped scripts.
