# src/xrGame/Weapon.h

> Declares the weapon base implemented across [`Weapon.cpp`](Weapon.cpp.md), [`WeaponFire.cpp`](WeaponFire.cpp.md), [`WeaponDispersion.cpp`](WeaponDispersion.cpp.md) and [`WeaponUpgrade.cpp`](WeaponUpgrade.cpp.md).

**Needs** — [`ShootingObject.h`](ShootingObject.h.md) · [`hud_item_object.h`](hud_item_object.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`firedeps.h`](firedeps.h.md) · [`CameraRecoil.h`](CameraRecoil.h.md) · [`first_bullet_controller.h`](first_bullet_controller.h.md) · [`Actor_Flags.h`](Actor_Flags.h.md) · [`PHShellCreator.h`](PHShellCreator.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — [`ActorAnimation.cpp`](ActorAnimation.cpp.md) · [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorInput.cpp`](ActorInput.cpp.md) · [`Actor_Events.cpp`](Actor_Events.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`CustomDetector.cpp`](CustomDetector.cpp.md) · [`EffectorShot.cpp`](EffectorShot.cpp.md) · [`Explosive.cpp`](Explosive.cpp.md) · [`Inventory.cpp`](Inventory.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](Level_bullet_manager_firetrace.cpp.md) · [`UIGameCTA.cpp`](UIGameCTA.cpp.md) · [`Weapon.cpp`](Weapon.cpp.md) · [`WeaponAmmo.cpp`](WeaponAmmo.cpp.md) · [`WeaponDispersion.cpp`](WeaponDispersion.cpp.md) · _and 43 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeapon`, the common ancestor of every firearm and melee weapon. Its
implementation is deliberately spread over four files — the bulk and the lifecycle in
[`Weapon.cpp`](Weapon.cpp.md), the shot itself in [`WeaponFire.cpp`](WeaponFire.cpp.md),
the accuracy model in [`WeaponDispersion.cpp`](WeaponDispersion.cpp.md), and the
configuration-driven upgrade installers in
[`WeaponUpgrade.cpp`](WeaponUpgrade.cpp.md) — because the four concerns were maintained
by different people. The split is not structural and a rebuild may merge them.

The header is also where the weapon *state* vocabulary is fixed, which the whole
`Weapon*` family shares:

- **States** extend the held-item states with: fire, secondary fire, reload, misfire,
  magazine-empty and switch.
- **Reload sub-states**: begin, in progress, end — used only by weapons that reload one
  round at a time (see [`WeaponShotgun.cpp`](WeaponShotgun.cpp.md)).
- **Addon status**, per addon slot: disabled, attachable, permanent.
- The sentinel `undefined_ammo_type` (all bits set of an 8-bit value) means "no pending
  ammo type change".

Exported units, by group:

**Lifecycle** — `Load`, `net_Spawn`, `net_Destroy`, `net_Export`, `net_Import`, `save`,
`load`, `OnEvent`, `reinit`, `reload`, `UpdateCL`, `shedule_Update`, and the four
parent-attachment hooks.

**Placement** — `UpdateXForm`, `UpdatePosition`, `UpdateFireDependencies`, the four
fire-point accessors (`get_LastFP`, `get_LastFP2`, `get_LastFD`, `get_LastSP`),
`get_ParticlesXFORM`, `ForceUpdateFireParticles`, `UpdateHudAdditonal`.

**Firing** — `FireStart`, `FireEnd`, `FireTrace`, `StopShooting`, `Reload`, `OnShot`,
`CheckForMisfire`, `IsMisfire`, and the four shot-effector hooks.

**Accuracy** — `GetBaseDispersion`, `GetFireDispersion` (two forms),
`GetConditionDispersionFactor`, `GetConditionMisfireProbability`, the five per-shot
dispersion model accessors, `GetCrosshairInertion`, `GetFirstBulletDisp`, and the two
camera recoil rigs.

**Ammunition** — `GetAmmoElapsed`, `GetAmmoMagSize`, `GetSuitableAmmoTotal`,
`GetAmmoCount`, `GetAmmoCount_forType`, `SetAmmoElapsed`, `SwitchAmmoType`, `SpawnAmmo`,
`OnMagazineEmpty`, `unlimited_ammo`, `GetMagazineWeight`, `IsNecessaryItem`,
`m_ammoTypes` and `m_magazine` (public, because the firing code and the heads-up display
both read them directly).

**Addons** — the three `Is*Attached` predicates, the three `*Attachable` predicates, the
three status accessors, `InitAddons`, `UpdateAddonsVisibility`,
`UpdateHUDAddonsVisibility`, the icon-offset and name accessors used by the buy and
upgrade screens, and `GetAddonsState`/`SetAddonsState`.

**Zoom** — `IsZoomEnabled`, `IsZoomed`, `OnZoomIn`, `OnZoomOut`, `ZoomInc`, `ZoomDec`,
`CurrentZoomFactor`, `GetZoomFactor`/`SetZoomFactor`, `IsRotatingToZoom`, `ZoomTexture`,
`ZoomHideCrosshair`, `GetCurrentHudOffsetIdx`, `UseScopeTexture`,
`EnableActorNVisnAfterZoom`.

**AI and planner** — `can_kill` (three forms), `ready_to_kill`, `hit_probability`,
`ef_main_weapon_type`, `ef_weapon_type`, `modify_holder_params`, `ParentIsActor`,
`ParentMayHaveAimBullet`, `NeedToDestroyObject`, `TimePassedAfterIndependant`.

**Rendering** — `renderable_Render`, `render_hud_mode`, `need_renderable`,
`render_item_ui`, `render_item_ui_query`, `show_crosshair`, `show_indicators`.

**Upgrades** — `install_upgrade_impl` and the four private installers for ammunition
class, dispersion, hit power and addons.

**Type tests** — `cast_weapon` and `cast_weapon_magazined`, the two downcasts the rest of
the game uses instead of run-time type queries. A rebuild with a real type system should
delete both.

Note `m_sub_state` and `m_ammoTypes` are public only because callers outside the
hierarchy reach them; they are logically protected.
