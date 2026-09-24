# src/xrGame/smart_cover_object.h

> Declares the placed entity that carries a smart cover: an invisible, non-updating volume whose only jobs are to own a transform, own a shape, and hand the AI a cover.

**Needs** — [`GameObject.h`](GameObject.h.md) · [`smart_cover.h`](smart_cover.h.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`cover_manager.cpp`](cover_manager.cpp.md) · [`script_game_object_inventory_owner.cpp`](script_game_object_inventory_owner.cpp.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`smart_cover.cpp`](smart_cover.cpp.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.cpp`](smart_cover_description.cpp.md) · [`smart_cover_object.cpp`](smart_cover_object.cpp.md) · [`smart_cover_object_inline.h`](smart_cover_object_inline.h.md) · [`smart_cover_object_script.cpp`](smart_cover_object_script.cpp.md)
**Tier floor** — T2: an entity with a collision shape but no per-frame work

## Purpose

Declares the surface implemented in
[`smart_cover_object.cpp`](smart_cover_object.cpp.md),
[`smart_cover_object_inline.h`](smart_cover_object_inline.h.md) and
[`smart_cover_object_script.cpp`](smart_cover_object_script.cpp.md).

This is a [client object](../../GLOSSARY.md) in the ordinary sense — it is spawned from a
[server object](../../GLOSSARY.md) record like any entity — but almost every behaviour an
entity normally has is switched off. It does not update, does not render, is not felt by
touch, cannot be used, is not visible to [zones](../../GLOSSARY.md), and is not an
obstacle. What it does have is a transform (which the cover's local geometry is resolved
through) and a shape (which defines "inside this cover").

## Exported units

- **The class** — a game object that owns a placed cover.
- **`net_Spawn`** — build the shape from the server record, read the two enemy-distance
  thresholds, register the cover with the cover manager, then disable itself.
- **`inside`** — is a world position within any of the shape's volumes.
- **`Center` / `Radius`** — the shape's bounding sphere in world space; the AI's spatial
  queries use these.
- **`enter_min_enemy_distance` / `exit_min_enemy_distance`** — the asymmetric thresholds
  the loophole scoring in [`smart_cover.cpp`](smart_cover.cpp.md) applies.
- **`get_cover`** — the placed cover; asserts there is one.
- **The behaviour opt-outs** — each answers a question the entity framework asks, and the
  set of answers is the real content of this declaration: uses navigation locations, does
  *not* inherit them from a parent, does not register with the
  [scheduler](../../GLOSSARY.md), does not validate its position on spawn, is not an
  obstacle, is not visible to zones, refuses touch and refuses use.
- **`OnRender`** — development-build visualization of the shape and each loophole's arc.
- **Script registration** — see
  [`smart_cover_object_script.cpp`](smart_cover_object_script.cpp.md).

## Notes

`UpdateCL` and the scheduled update are declared and then made unreachable in the
implementation. That is the strongest possible statement of the design: this entity must
never be updated, and reaching either is a bug in whatever registered it. A rebuild should
express it as the entity simply not implementing the update interface.
