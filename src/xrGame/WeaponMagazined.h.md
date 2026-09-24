# src/xrGame/WeaponMagazined.h

> Declares the firing state machine implemented in [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md).

**Needs** — [`Weapon.h`](Weapon.h.md) · [`HudSound.h`](HudSound.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md)
**Used by** — [`Actor_Movement.cpp`](Actor_Movement.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`EffectorZoomInertion.cpp`](EffectorZoomInertion.cpp.md) · [`EffectorZoomInertion.h`](EffectorZoomInertion.h.md) · [`WeaponAutomaticShotgun.cpp`](WeaponAutomaticShotgun.cpp.md) · [`WeaponAutomaticShotgun.h`](WeaponAutomaticShotgun.h.md) · [`WeaponCustomPistol.cpp`](WeaponCustomPistol.cpp.md) · [`WeaponCustomPistol.h`](WeaponCustomPistol.h.md) · [`WeaponFN2000.cpp`](WeaponFN2000.cpp.md) · [`WeaponFN2000.h`](WeaponFN2000.h.md) · [`WeaponLR300.cpp`](WeaponLR300.cpp.md) · [`WeaponLR300.h`](WeaponLR300.h.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) · _and 19 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares `CWeaponMagazined`, the base of every firearm that feeds from a magazine.
Substance is in [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md).

The header fixes one constant worth naming: a negative burst length means "fire until the
magazine is empty". There is no separate automatic flag.

It also fixes the shape subclasses extend: a set of `switch2_*` procedures, one per
state, called from a single transition table; a set of `PlayAnim*` procedures, one per
animation slot; and `state_*` procedures for the three states that do per-frame work
(fire, misfire, magazine-empty). A subclass changing one aspect of weapon behaviour
overrides one member of one of those three families, which is why the dozen concrete
weapon classes in this directory are each a few dozen lines.

Exported units, by group:

**Construction and configuration** — `CWeaponMagazined(sound_type)` (the sound-type
argument tags every one of this weapon's sounds for the AI's hearing system),
`Load`, `LoadSilencerKoeffs`, `SetDefaults`, `install_upgrade_impl`.

**Firing** — `FireStart`, `FireEnd`, `Reload`, `TryReload`, `state_Fire`,
`state_Misfire`, `state_MagEmpty`, `OnShot`, `OnEmptyClick`, `FireBullet`,
`GetFireDispersion`, `GetWeaponDeterioration`, `ShotsFired`, `AllowFireWhileWorking`.

**State machine** — `OnStateSwitch`, `OnAnimationEnd`, and the seven `switch2_*`
procedures (idle, fire, empty, reload, hiding, hidden, showing).

**Ammunition** — `ReloadMagazine`, `UnloadMagazine`, `OnMagazineEmpty`,
`IsAmmoAvailable`.

**Addons** — `Attach`, `Detach`, `DetachScope`, `CanAttach`, `CanDetach`, `InitAddons`,
`ApplySilencerKoeffs`, `ResetSilencerKoeffs`.

**Fire modes** — `SwitchMode`, `SingleShotMode`, `SetQueueSize`, `GetQueueSize`,
`OnNextFireMode`, `OnPrevFireMode`, `HasFireModes`, `GetCurrentFireMode`,
`StopedAfterQueueFired`.

**Presentation** — `UpdateCL`, `UpdateSounds`, `GetBriefInfo`, `OnZoomIn`, `OnZoomOut`,
and the seven `PlayAnim*`/`PlayReloadSound` procedures.

**Lifecycle** — `net_Destroy`, `net_Export`, `net_Import`, `save`, `load`,
`OnH_A_Chield`, `Action`.

`GetCurrentFireMode` answers 1 when the weapon has no authored mode list, because the
original assumed every magazined weapon had one and the shipped data does not.
