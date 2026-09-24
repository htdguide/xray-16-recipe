# src/xrGame/WeaponMagazinedWGrenade.h

> Declares the rifle with an under-barrel grenade launcher, implemented in [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md).

**Needs** — [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`RocketLauncher.h`](RocketLauncher.h.md)
**Used by** — [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`WeaponAK74.cpp`](WeaponAK74.cpp.md) · [`WeaponAK74.h`](WeaponAK74.h.md) · [`WeaponGroza.cpp`](WeaponGroza.cpp.md) · [`WeaponGroza.h`](WeaponGroza.h.md) · [`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md) · [`WeaponScript.cpp`](WeaponScript.cpp.md) · [`game_cl_deathmatch_buywnd.cpp`](game_cl_deathmatch_buywnd.cpp.md) · [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md) · [`player_hud.cpp`](player_hud.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`UIHudStatesWnd.cpp`](ui/UIHudStatesWnd.cpp.md) · [`UIMpTradeWnd_items.cpp`](ui/UIMpTradeWnd_items.cpp.md) · [`UIMpTradeWnd_wpn.cpp`](ui/UIMpTradeWnd_wpn.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponMagazinedWGrenade`, two weapons in one object. Substance is in
[`WeaponMagazinedWGrenade.cpp`](WeaponMagazinedWGrenade.cpp.md).

The declaration's own load-bearing content is the **shadow ammunition set** — a second
type list, selected type, magazine, capacity, prototype cartridge and elapsed count,
which `PerformSwitchGL` exchanges with the inherited ones. They are public because the
heads-up display and the upgrade installers reach them directly.

Note the capacity field's name is misleading: `iMagazineSize2` holds the **rifle's**
capacity at all times, because the launcher's is always one.

Exported units, by group:

**Mode switching** — `SwitchMode` (the user-visible toggle: guard, swap, sound,
animation), `CanSwitchToGL` (settled state, not pending, launcher attached),
`PerformSwitchGL` (the raw field exchange), `PlayAnimModeSwitch`.

**Grenade firing** — `LaunchGrenade` (the ballistic solve and launch from the second
muzzle), `Action` (intercepts the fire binding in grenade mode), `OnEvent` (rocket
ownership; the launch event plays the shot's effects), `state_Fire` (empty in grenade
mode), `FireEnd`, `OnMagazineEmpty`, `OnShot`, `switch2_Reload`, `ReloadMagazine`.

**Addons** — `CanAttach`, `CanDetach`, `Attach`, `Detach` (the swap-unload-swap dance),
`InitAddons` (re-read the launch speed from the attached launcher).

**Presentation** — `UseScopeTexture` and `CurrentZoomFactor` (the launcher has its own
sight), `GetCurrentHudOffsetIdx` (a third authored hand pose),
`UpdateGrenadeVisibility`, `UpdateSounds`, `GetBriefInfo` (adds the grenade reserve
field), and the seven `PlayAnim*` overrides with their three variants each.

**Lifecycle** — `Load`, `net_Spawn`, `net_Destroy`, `net_Export`/`net_Import` (mode flag
first), `save`/`load`, `OnStateSwitch`, `OnAnimationEnd`, `OnH_B_Independent`.

**Other** — `Weight` (adds the shadow magazine), `IsNecessaryItem` (either type list),
`GetAmmoCount2`, and the two upgrade installers.
