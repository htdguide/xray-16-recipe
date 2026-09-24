# src/xrGame/Grenade.h

> Declares the thrown grenade, implemented in [`Grenade.cpp`](Grenade.cpp.md).

**Needs** — [`Missile.h`](Missile.h.md) · [`Explosive.h`](Explosive.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`Actor_Feel.cpp`](Actor_Feel.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`F1.h`](F1.h.md) · [`Grenade.cpp`](Grenade.cpp.md) · [`HitMarker.cpp`](HitMarker.cpp.md) · [`HitMarker.h`](HitMarker.h.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`PhysicsShellHolder.cpp`](PhysicsShellHolder.cpp.md) · [`RGD5.h`](RGD5.h.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`inventory_quickswitch.cpp`](inventory_quickswitch.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CGrenade` as the combination of a missile (a throwable held item with a
first-person animation state machine) and an explosive (a fuse, a blast and a damage
source). Substance is in [`Grenade.cpp`](Grenade.cpp.md).

Exported units:

- `CGrenade` — the item.
- `Load`, `net_Spawn`, `net_Destroy`, `net_Relcase`, `OnEvent`, `UpdateCL` — the
  lifecycle, each forwarding to both bases.
- `Throw`, `DropGrenade`, `Destroy` — launch the fake missile, launch it because
  something forced the issue, and detonate.
- `State`, `OnAnimationEnd`, `DiscardState`, `DeactivateItem`, `SendHiddenItem` — the
  animation state machine's hooks, including the two paths that must not lose a primed
  grenade.
- `PutNextToSlot` — promote the next grenade of the stack into the hand.
- `Hit` — cook-off from a nearby explosion.
- `Action` — the "next weapon" command, repurposed to cycle grenade kinds.
- `Useful`, `NeedToDestroyObject`, `TimePassedAfterIndependant` — whether the alife
  simulation keeps it, and the ground-litter timeout.
- `GetBriefInfo` — the HUD summary, whose "ammo" is the stack count.
- `OnH_A_Chield`, `OnH_A_Independent`, `OnH_B_Independent`, `OnH_B_Chield` — parentage
  transitions that start and cancel the decay clock.
- `set_destroy_callback` — a one-shot notification for whoever needs to know this
  grenade is gone, fired by both the destruction and the detonation paths.
- The capability casts: it is an explosive, a missile, a HUD item, a game object and a
  damage source, the last of which it answers on the explosive half's behalf.

## Notes

The header inherits the touch sense but the implementation never uses it; it comes in
through the explosive base, which needs to know what is inside the blast.
