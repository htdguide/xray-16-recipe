# src/xrGame/physic_item.h

> Declares the simplest physical object — a thing that falls, bounces and can be carried — implemented in [`physic_item.cpp`](physic_item.cpp.md).

**Needs** — [`GameObject.h`](GameObject.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`PHShellCreator.h`](PHShellCreator.h.md) · [`physic_item_inline.h`](physic_item_inline.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`CustomRocket.cpp`](CustomRocket.cpp.md) · [`CustomRocket.h`](CustomRocket.h.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`eatable_item.cpp`](eatable_item.cpp.md) · [`eatable_item_object.cpp`](eatable_item_object.cpp.md) · [`eatable_item_object.h`](eatable_item_object.h.md) · [`inventory_item_object.cpp`](inventory_item_object.cpp.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`physic_item.cpp`](physic_item.cpp.md) · [`player_hud.cpp`](player_hud.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CPhysicItem`, the base for every object that is a rigid body when loose in the world
and nothing when carried: thrown bolts, dropped weapons, debris. Substance is in
[`physic_item.cpp`](physic_item.cpp.md).

Exported units:

- `CPhysicItem` — a physics-shell holder that also knows how to build simple shells. Holds one
  flag: whether it is already committed to being destroyed.
- `net_Spawn` — build a shell only if the item spawned loose rather than inside someone.
- `OnH_B_Independent` / `OnH_B_Chield` — the two ownership transitions: becoming loose turns
  the body on and makes the item visible, being picked up turns both off.
- `UpdateCL` — per-frame: take the interpolated body transform as the item's transform.
- `activate_physic_shell` / `setup_physic_shell` — bring a body into existence, from a parent's
  transform or from the item's own.
- `create_box_physic_shell` / `create_box2sphere_physic_shell` / `create_physic_shell` — the
  three shell shapes an item can be given.
