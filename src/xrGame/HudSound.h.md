# src/xrGame/HudSound.h

> Declares the alias-keyed sound bank held items play from, implemented in [`HudSound.cpp`](HudSound.cpp.md).

**Needs** — [`xrSound/Sound.h`](../xrSound/Sound.h.md)
**Used by** — [`CarWeapon.cpp`](CarWeapon.cpp.md) · [`CarWeapon.h`](CarWeapon.h.md) · [`CustomDetector.h`](CustomDetector.h.md) · [`Helicopter.cpp`](Helicopter.cpp.md) · [`HelicopterWeapon.cpp`](HelicopterWeapon.cpp.md) · [`HudItem.cpp`](HudItem.cpp.md) · [`HudItem.h`](HudItem.h.md) · [`HudSound.cpp`](HudSound.cpp.md) · [`Missile.h`](Missile.h.md) · [`Torch.cpp`](Torch.cpp.md) · [`Torch.h`](Torch.h.md) · [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md) · [`WeaponBinocularsVision.h`](WeaponBinocularsVision.h.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md) · _and 5 more_
**Tier floor** — T3: a declaration only

## Purpose

Declares the three tiers of the sound bank. Substance is in
[`HudSound.cpp`](HudSound.cpp.md).

Exported units:

- `HUD_SOUND_ITEM` — one alias and its interchangeable takes, each with its own gain and
  start delay, plus whether the alias is exclusive and which take is currently active.
  - `LoadSound` (two forms: one take into a raw handle, or a whole numbered take list),
    `DestroySound`, `PlaySound`, `StopSound` — static, taking the item by reference.
  - `playing`, `set_position` — the active take's state, with the position setter
    dropping a non-positional take because it cannot be moved.
  - Equality against a text alias, which is how a bank looks one up.
- `HUD_SOUND_COLLECTION` — a bank of aliases.
  - `LoadSound` — add an alias, optionally exclusive.
  - `PlaySound`, `StopSound`, `SetPosition`, `StopAllSounds` — play tolerates an unknown
    alias, stop does not.
  - `FindSoundItem` — the lookup, with a flag choosing between asserting and reporting.
  - `m_alias` — set only when this bank is one layer of a layered bank.
- `HUD_SOUND_COLLECTION_LAYERED` — several banks sharing one alias, played as one sound.
  - `LoadSound` (two forms, global or from a supplied configuration), which recognises
    both an authored multi-layer section and a legacy single sound.
  - `PlaySound`, `StopSound`, `SetPosition`, `StopAllSounds`, `FindSoundItem`.

## Notes

Both collections' item lists are public in the original, and code outside reaches in to
iterate them. Treat them as the module's own state in a rebuild; the exposure is
incidental.
