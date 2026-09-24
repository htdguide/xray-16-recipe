# src/xrGame/ai/monsters/flesh/flesh.h

> Declares the flesh, implemented in [`flesh.cpp`](flesh.cpp.md).

**Needs** — [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md) · [`flesh.cpp`](flesh.cpp.md)
**Used by** — [`flesh.cpp`](flesh.cpp.md) · [`flesh_script.cpp`](flesh_script.cpp.md) · [`flesh_state_manager.cpp`](flesh_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the flesh as a base creature that is additionally controllable — a controller can take it
over. The class adds almost nothing to its base, which is the point; see
[`flesh.cpp`](flesh.cpp.md).

## `CAI_Flesh`

- **Load** — build the animation vocabulary
- **net_Spawn** — pure forwarding; carries no decision
- **CheckSpecParams** — the one-off animation flourishes: drag, inspect a corpse, attack from
  behind, threaten
- **ability_can_drag** — yes; a flesh drags corpses
- **get_monster_class_name** — `"flesh"`
- **ConeSphereIntersection** — a private, unused cone-versus-sphere predicate; see the
  implementation twin for why it is worth knowing about

Also declared: the script registration hook (see [`flesh_script.cpp`](flesh_script.cpp.md)).
