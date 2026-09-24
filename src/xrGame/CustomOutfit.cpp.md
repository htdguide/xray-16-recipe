# src/xrGame/CustomOutfit.cpp

> The body armour: it wears out as it absorbs damage, reduces incoming damage by type and by which body part was struck, changes the wearer's appearance and first-person arms, and modifies carrying capacity and regeneration.

**Needs** — [`CustomOutfit.h`](CustomOutfit.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`BoneProtections.h`](BoneProtections.h.md) · [`Inventory.h`](Inventory.h.md) · [`Actor.h`](Actor.h.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`Torch.h`](Torch.h.md) · [`Level.h`](Level.h.md) · [`player_hud.h`](player_hud.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: damage arithmetic plus model and asset swaps

## Purpose

An outfit is the single largest modifier on how much damage the player takes, and this
file is almost entirely the damage formula. What makes it worth a long page is that there
are **three** damage formulas, selected per outfit, because the three shipped games
computed armour differently and the engine loads all three games' data unmodified.

The three are not variants of one idea:

- **Clear Sky's formula** scales the incoming damage by how far the round's penetration
  exceeded the armour, floored at a configured fraction;
- **Call of Pripyat's formula** does the same but *only in multiplayer*, and in single
  player leaves a penetrating round entirely unreduced;
- **the third formula**, present for completeness, subtracts armour from damage rather
  than scaling it.

The engine has no field telling it which game's data it is reading, so it *infers* the
formula from the presence of two configuration keys, and the inference is documented in
the source as a hack. A rebuild should carry a real version marker and keep all three
formulas.

The second decision worth stating: **a bullet is resisted by the bone it hits, everything
else by the outfit as a whole.** Penetrating rounds consult a per-bone armour table and the
round's own armour-piercing value; every other damage type consults only the outfit-wide
per-type coefficient. That asymmetry is why the armour table is loaded against the
*wearer's skeleton*, which is why it has to be reloaded whenever the wearer changes.

## State

```text
RECORD Outfit
  hit_type_protection : list<real> per damage type   # 1.0 = no reduction
  bone_protection     : per-bone armour, per-bone protection factor,
                        per-bone "bullets pass straight through",
                        a hit fraction, and which of the three formulas to use
  bones_section       : text       # which bone table to load against the wearer
  actor_visual        : optional<text>    # the model the wearer is swapped to
  full_icon_name      : text              # the large inventory image
  nightvision_sect    : optional<text>    # night-vision parameters this outfit grants
  power_loss          : real in [0,1]     # stamina drain multiplier
  additional_weight   : real              # carrying capacity bonus
  additional_weight2  : real              # a second bonus; see Notes
  health/radiation/satiety/power/bleeding restore speeds : real
  artefact_count      : int in [0,5]      # artefact containers this outfit provides
  helmet_available    : bool
  equipment_type      : int               # the AI evaluation layer's classification
```

Invariants:

- every protection coefficient defaults to 1 (no reduction) and is multiplied by the
  outfit's **condition**, so a ruined outfit protects against nothing. Condition is the
  single number that makes armour a consumable.
- the artefact container count is clamped to five and the stamina multiplier to the unit
  range, whatever the data says.
- the per-bone table is meaningless without the wearer's skeleton: it maps bone names to
  values, and bone indices differ per model.

## `Load`

**Contract** — reads the eight per-type protection coefficients, the two derived ones, the
model and icon names, the stamina and carry modifiers, the five regeneration rates, the
bone table's section name, and the helmet permission. Two of the coefficients are derived
rather than read:

```text
physic_strike_protection : defaults to the strike coefficient
light_burn_protection    : always equals the burn coefficient, never read
fire_wound_protection    : read, but NOT used in the damage path — the bone table is used
                           instead. It exists as a number to show in the interface and for
                           scripts to reason about.
```

And the formula inference:

```text
IF the section declares a hit fraction THEN
  hit_fraction = that value
  IF the section ALSO declares fire_wound_protection THEN
    formula = Clear Sky's
  ELSE
    formula = Call of Pripyat's
```

**Notes** — the inference is unsound for modified data, which the original says outright:
a modification that adds the obsolete key to Call of Pripyat data silently switches the
whole game to the older damage model.

## `HitThroughArmor` — the damage formula

**Contract** — the central operation. Given incoming damage, the struck bone, the round's
armour-piercing value and the damage type, returns the damage that gets through, reports
whether a bleeding wound should be opened, and wears the outfit down. Runs one of three
formulas.

**Call of Pripyat's formula:**

```text
IF the damage is a penetrating round THEN
  armour = bone_armour(bone)
  IF armour is negative THEN RETURN damage unchanged     # this bone is unarmoured
  armour = armour * condition
  IF piercing > armour THEN
    # penetrated
    IF multiplayer THEN
      fraction = max((piercing - armour) / piercing, hit_fraction)
      damage = damage * fraction * bone_protection(bone)
    # in SINGLE PLAYER, a penetrating round is not reduced at all
  ELSE
    # stopped by the plate
    damage = damage * hit_fraction
    open no wound
ELSE
  weight = 1.0 for strike, wound, wound_2 and explosion; 0.1 for everything else
  damage = max(damage - outfit_protection(type) * weight, 0)

wear the outfit by the ORIGINAL damage
```

**Clear Sky's formula** differs in four ways, each of which matters:

```text
- the penetration reduction applies in single player too;
- the floor is applied to the whole multiplied damage rather than to the fraction,
  so it is a floor on absolute damage rather than on the ratio;
- the per-bone protection factor is applied only in multiplayer;
- strike damage is NOT in the full-weight group, so a melee blow is resisted
  at a tenth of the coefficient;
- the outfit is worn by the RESULTING damage, not the original — so armour that
  stops a blow completely takes no wear from it.
```

**The third formula** subtracts rather than scales: armour reduced by the round's piercing
is subtracted from the damage, floored at a fraction of the original, and non-penetrating
damage simply has the per-bone protection subtracted.

**Invariants** — the "no wound" flag is only ever cleared, never set, and only on the
stopped-by-armour path. Armour can prevent bleeding; nothing can cause it.

**Notes** — the difference in what the outfit is worn *by* — original damage in two
formulas, resulting damage in the third — is the difference between armour that degrades
from being shot and armour that degrades from failing to stop being shot. Both are
defensible; they are different games.

## `Hit`

**Contract** — wearing the outfit down. The damage is scaled by the outfit's own immunity to
that damage type — a separate table from the protection coefficients — and subtracted from
its condition.

## `GetDefHitTypeProtection` / `GetHitTypeProtection` / `GetBoneArmor`

**Contract** — three views of protection. The first is the outfit-wide coefficient scaled by
condition. The second additionally multiplies by the struck bone's protection factor. The
third is the raw per-bone armour value, unscaled — its callers apply condition themselves.

## `ReloadBonesProtection` / `AddBonesProtection`

**Contract** — bind the named bone table to a skeleton. The skeleton is the *wearer's*, and
which wearer that is depends on the game mode: in single player it is always the entity the
player is currently controlling, and in multiplayer it is whoever the outfit's parent is.
`AddBonesProtection` merges a second table on top, which is how an upgrade adds armour to
specific parts without replacing the whole table.

**Invariants** — the reload happens at two different times in the two modes: at spawn in
single player, and when the outfit is attached to a carrier in multiplayer. Both are
necessary; in single player the outfit may be spawned before any carrier exists, and in
multiplayer the carrier's model is not known until attachment.

**Notes** — using the *current view entity* rather than the parent in single player means an
outfit lying on the ground still binds its table to the player's skeleton. That is
harmless and slightly wasteful, and it is the simplest correct answer given that the player
is the only thing that wears one.

## `OnMoveToSlot` — putting it on

**Contract** — dressing the wearer. Swaps the model and the first-person arms, and then
enforces the helmet rule: an outfit that includes its own helmet does not permit a separate
one, so any worn helmet is moved to the backpack and the torch's night-vision mode is
re-applied — because the night vision that was the helmet's is now the outfit's.

```text
FUNCTION on_equip(previous_place)
  IF the wearer is the player THEN
    apply the skin model and the first-person arms
    IF coming from another slot AND this outfit has no separate helmet THEN
      re-enable the torch's night vision      # ownership of it moved to this outfit
    IF a helmet is worn AND this outfit has no separate helmet THEN
      move the helmet to the backpack
```

## `OnMoveToRuck` — taking it off

**Contract** — restores the wearer's default model and default first-person arms, and
switches the torch's night vision off if this outfit was the thing providing it.

## `ApplySkinModel`

**Contract** — the model swap, in both directions, with a multiplayer special case. Dressing
normally uses the outfit's own configured model; in multiplayer, if the wearer's *team* has
a skin declared for this outfit's section, that skin's path is composed and used instead —
so the same outfit looks different on each team. Undressing restores the wearer's recorded
default model.

Separately, and only when the wearer is the entity the player is looking through, the
first-person arms are reloaded: the outfit's own arms section if it declares one, the
default otherwise. This is why gloves change when you change suits.

**Invariants** — the arms reload is gated on the wearer being the *view* entity, not on
being the player. A player spectating or possessing another creature gets that creature's
arms.

## `GetPowerLoss`

**Contract** — the stamina drain multiplier, with a wrinkle: under the two older formulas, a
*ruined* outfit stops draining stamina at all — its multiplier snaps back to 1. Under Call
of Pripyat's formula the multiplier applies regardless of condition.

**Notes** — the source apologises for using the damage-formula marker to decide an
unrelated question, which is exactly the cost of inferring the data's generation rather
than recording it.

## `install_upgrade_impl`

**Contract** — the workbench upgrade path. An upgrade section may override any protection
coefficient, the night-vision section, the bone table (which triggers a reload), an
*additional* bone table (which is merged on top), the hit fraction (only under the two
formulas that use one), both carry bonuses, all five regeneration rates, the stamina
multiplier and the container count. Every field is optional; the clamps are re-applied
afterwards. A test pass answers whether the upgrade would change anything without changing
it.

**Invariants** — the two bone-table keys behave differently and deliberately: one replaces,
one adds. An upgrade that armours the shoulders uses the additive form so it composes with
whatever the outfit already had.

## `net_Export` / `net_Import`

**Contract** — the only replicated state is the condition, quantized to a byte over the unit
range. Everything else is derivable from the section.

## `BonePassBullet` / `ef_equipment_type` / `GetFullIconName` / `get_artefact_count`

**Contract** — small queries: whether a named bone lets bullets through untouched, the AI
evaluation layer's classification of this equipment, the large inventory image, and how
many artefact containers the outfit provides.

## Could not recover

- Two separate carrying-capacity bonuses exist with no recorded distinction between them.
  Reading the consumers is the only way to tell them apart; nothing here says.
- The light-burn coefficient is always assigned from the burn coefficient and has no key
  of its own, so the two can never differ.
- `fire_wound_protection` is read, exported to scripts and shown in the interface, and is
  not part of any damage calculation.
- The armour-value sign convention — a negative per-bone armour meaning "this bone is
  unarmoured, pass the damage through" — appears only in the Call of Pripyat branch; the
  other two formulas treat a negative armour as arithmetic.
