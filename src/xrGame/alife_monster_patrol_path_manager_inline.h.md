# src/xrGame/alife_monster_patrol_path_manager_inline.h

> Accessors of the offline patrol cursor — mostly trivial, but one of them carries a real rule.

**Needs** — [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md)
**Used by** — [`alife_monster_patrol_path_manager.h`](alife_monster_patrol_path_manager.h.md)
**Tier floor** — T3: field reads and writes

## Purpose

Bodies for the always-inlined accessors of `CALifeMonsterPatrolPathManager`, split out of
the header as a C++ habit. A rebuild folds them into the type. Substance is in
[`alife_monster_patrol_path_manager.cpp`](alife_monster_patrol_path_manager.cpp.md).

Two of these are not simple field access and must survive a rebuild.

## `path` (setter)

**Contract** — installs a patrol path and decides whether the existing cursor survives.

```text
FUNCTION set_path(new_path)
  actual = actual AND (current_path == new_path)
  current_path = new_path
```

**Notes** — re-assigning the *same* path is a no-op for the cursor; assigning a different
one (or clearing it) invalidates the cursor so the next update re-joins from the
configured start type. This is what makes it safe for a smart terrain to restate a
creature's patrol job on every tick — a naive setter that always invalidated would pin
the creature to its joining point forever. Note that the invalidation is one-way: the
flag is never re-raised here, only in `actualize`.

## `completed`

**Contract** — reports finished only when the cursor is also current: `completed AND
actual`. A path swap therefore un-finishes the creature without any explicit reset,
which is why the finished flag is cleared in `actualize` rather than in the setter.

## The remaining accessors

**Contract** — `object` and `path` (getter) return the owner and the installed path by
reference, both required to be present. `start_type`, `route_type`, `use_randomness` and
`start_vertex_index` are plain get/set over the four traversal settings; `actual` reads
the cursor-valid flag.
