# src/xrGame/EntityCondition.h

> Declares what a living creature's condition is made of — the value set, the wound list, the seventeen boost kinds and the consumable record — implemented in [`EntityCondition.cpp`](EntityCondition.cpp.md).

**Needs** — [`hit_immunity.h`](hit_immunity.h.md) · [`game_type.h`](game_type.h.md)
**Used by** — [`ActorCondition.cpp`](ActorCondition.cpp.md) · [`ActorCondition.h`](ActorCondition.h.md) · [`ActorCondition_script.cpp`](ActorCondition_script.cpp.md) · [`BottleItem.cpp`](BottleItem.cpp.md) · [`CustomMonster.cpp`](CustomMonster.cpp.md) · [`Entity.cpp`](Entity.cpp.md) · [`Entity.h`](Entity.h.md) · [`EntityCondition.cpp`](EntityCondition.cpp.md) · [`base_monster_misc.cpp`](ai/monsters/basemonster/base_monster_misc.cpp.md) · [`monster_state_rest.h`](ai/monsters/states/monster_state_rest.h.md) · [`eatable_item.cpp`](eatable_item.cpp.md) · [`entity_alive.cpp`](entity_alive.cpp.md) · [`entity_alive.h`](entity_alive.h.md) · [`entity_alive_inline.h`](entity_alive_inline.h.md) · _and 2 more_
**Tier floor** — T3: a declaration, plus one data table

## Purpose

Declares the condition model every living thing carries. Substance in
[`EntityCondition.cpp`](EntityCondition.cpp.md). The header is substantive in one respect:
it carries the **boost vocabulary**, which is a frozen list of names that appear in the
shipped configuration files and in the user interface.

Exported units:

- `EBoostParams` and `ef_boosters_section_names` — the seventeen temporary modifiers a
  consumable may grant, and **the configuration key each one reads**. The two must stay in
  the same order; nothing enforces it, and both are indexed independently by different
  callers (the eatable-item code and the item-description screen). A rebuild should make
  this one table of (kind, key) pairs.

  The seventeen: health restore, stamina restore, radiation restore, bleeding restore,
  maximum carried weight, three *protections* (radiation, telepathic, chemical burn) and
  nine *immunities* (burn, shock, radiation, telepathic, chemical burn, explosion, strike,
  fire wound, wound). Protection is subtracted from a hit's power; immunity is subtracted
  from its multiplier — see [`EntityCondition.cpp`](EntityCondition.cpp.md).

- `SBooster` — one active modifier: a kind, a value and a remaining time. Its load reads the
  key its kind names.
- `SMedicineInfluenceValues` — what a consumable does immediately: health, stamina, satiety,
  dose, a fraction of wounds healed (clamped to zero-to-one), a maximum-stamina increase,
  alcohol, and an optional application time. An influence with a time applies gradually; one
  without applies at once.
- `CEntityConditionSimple` — health and maximum health alone. The split exists for things
  that are damageable but not alive.
- `CEntityCondition` — the full model. Also a hit-immunity carrier, which is where the
  per-damage-type multipliers live.
- `LoadCondition`, `LoadTwoHitsDeathParams`, `reinit` — configuration. The condition may name
  a *separate* section to read its rates from, so many creatures can share one tuning block.
- `ChangeHealth`, `ChangePower`, `ChangeRadiation`, `ChangePsyHealth`, `ChangeSatiety`,
  `ChangeAlcohol`, `ChangeBleeding`, `ChangeCircumspection`, `ChangeEntityMorale` — **the
  only ways to change a value**, and none of them writes one: each adds to an accumulator
  consumed by `UpdateCondition`. Satiety and alcohol are empty here and filled by the
  player's subclass.
- `ConditionHit` — apply a hit; the type table.
- `UpdateCondition`, `UpdateWounds`, `UpdateConditionTime`, `SetConditionDeltaTime` — the
  per-update pass and the clock that drives it.
- `AddWound`, `ClearWounds`, `BleedingSpeed`, `wounds` — the wound list, one per bone.
- `IsLimping` — stamina times health below a threshold.
- `CanBeHarmed`, `SetCanBeHarmedState` — the harm veto, which also requires being on the
  authoritative side.
- `ApplyInfluence`, `ApplyBooster` — consume a consumable.
- `save`, `load`, `remove_links` — persistence, and what happens when the recorded attacker
  is deleted.
- `SConditionChangeV` — the seven per-second rates, loaded with an optional key suffix so one
  section can hold several difficulty levels.
- `hit_bone_scale`, `wound_bone_scale`, `change_v`, `radiation` — writable references handed
  out so the damage manager can push per-bone scales in immediately before a hit and so
  scripts can retune rates at run time. They are a deliberate hole in the accumulator
  discipline; a rebuild should pass the scales with the hit instead.

**Notes** — `GetSatiety` on the base returns a constant one. Only the player has a satiety
model; everything else is permanently sated, which matters because the same user-interface
code reads it for any creature.
