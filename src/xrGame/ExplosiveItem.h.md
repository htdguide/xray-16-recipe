# src/xrGame/ExplosiveItem.h

> Declares the fused environmental explosive — canisters, gas bottles — implemented in [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md).

**Needs** — [`Explosive.h`](Explosive.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`DelayedActionFuse.h`](DelayedActionFuse.h.md)
**Used by** — [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md) · [`game_cl_mp.cpp`](game_cl_mp.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the object that explodes a configured time *after* being damaged rather than when it
is hit. Substance in [`ExplosiveItem.cpp`](ExplosiveItem.cpp.md).

Three parents: an inventory item (so it can be carried, has a condition, and takes damage
like an item), a **delayed-action fuse** (the countdown), and the explosion itself. The fuse
is the reason the class exists.

Exported units:

- `CExplosiveItem` — the object.
- `Load` — reads the item, the explosion and the fuse's two numbers from one section, and
  requires a warning effect to be named.
- `Hit` — light the fuse, record who lit it, and make the item immune to further damage while
  it burns.
- `shedule_Update`, `shedule_Needed` — run the countdown, and force the item to keep being
  scheduled while it burns even when nothing else would.
- `StartTimerEffects` — the visible hiss.
- `GetRayExplosionSourcePos` — the blast's sampling rays start inside the item's own box.
- `ActivateExplosionBox` — **deliberately empty**: this explosive does not push bodies apart
  with an expanding shape, because one at floor level would launch the item itself.
- `UpdateCL`, `OnEvent`, `net_Destroy`, `net_Relcase` — dispatch to both the explosion and the
  item.
- `net_Spawn`, `net_Export`, `net_Import` — the item's alone.
- `ChangeCondition` — forced to the inventory item's meaning of condition, not the explosive's.
- `cast_game_object`, `cast_explosive`, `cast_IDamageSource` — which face a caller gets.
