# src/xrGame/ai/monsters/fracture/fracture.h

> Declares the fracture, implemented in [`fracture.cpp`](fracture.cpp.md).

**Needs** — [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`fracture.cpp`](fracture.cpp.md)
**Used by** — [`fracture.cpp`](fracture.cpp.md) · [`fracture_script.cpp`](fracture_script.cpp.md) · [`fracture_state_manager.cpp`](fracture_state_manager.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Names the fracture as a plain base creature — not controllable, with no additional state and no
additional abilities. It is the chapter's minimal creature declaration.

## `CFracture`

- **Load** — build the animation vocabulary
- **CheckSpecParams** — the two flourishes: inspect a corpse, stand scared
- **get_monster_class_name** — `"fracture"`

Also declared: the script registration hook (see
[`fracture_script.cpp`](fracture_script.cpp.md)).
