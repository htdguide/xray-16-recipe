# src/xrGame/GrenadeLauncher.cpp

> The under-barrel grenade launcher attachment: an inventory item whose entire content is one tuned number, the muzzle velocity it gives a launched grenade.

**Needs** — [`GrenadeLauncher.h`](GrenadeLauncher.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — reached through its declarations in [`GrenadeLauncher.h`](GrenadeLauncher.h.md); callers name that, not this file.
**Tier floor** — T3: reads one configuration value; every other method is a pass-through

## Purpose

A weapon upgrade that exists as an inventory item in its own right, so that it can be
found, traded and attached. Its only behaviour is to carry the muzzle velocity that the
weapon it is attached to uses when launching a grenade; the launching itself belongs to
the weapon.

The file is almost entirely pass-throughs to the generic inventory item. That is not
padding in the original either — the overrides exist so the class has the right virtual
surface for the object factory, and a rebuild can delete every one of them.

## State

```text
RECORD GrenadeLauncher
  grenade_velocity : real   # initial speed given to a launched grenade, from the section
```

## `Load`

**Contract** — reads the muzzle velocity from the entity's configuration section, then
loads the generic inventory item. The velocity key is required; a section without it
fails to load.

**Notes** — the read happens *before* the base load rather than after, which is the
reverse of every other item in the chapter. Nothing depends on the order here, but a
rebuild should normalize it.

## `GetGrenadeVel`

**Contract** — the configured muzzle velocity. The attached weapon's only question of
this object.

## `net_Spawn`, `net_Destroy`, `UpdateCL`, `OnH_A_Chield`, `OnH_B_Independent`

**Contract** — pure delegation to the generic inventory item. Present only to complete
the virtual surface.
