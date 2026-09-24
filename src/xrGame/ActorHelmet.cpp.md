# src/xrGame/ActorHelmet.cpp

> The helmet: a worn item that reduces incoming damage per bone and per damage type, and whose damage formula differs depending on which of the three games' data it came from.

**Needs** — [`ActorHelmet.h`](ActorHelmet.h.md) · [`BoneProtections.h`](BoneProtections.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`Actor.h`](Actor.h.md) · [`Inventory.h`](Inventory.h.md) · [`Torch.h`](Torch.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: participates in the damage path and in the network state export

## Purpose

A helmet is an outfit that covers the head only. Its real content is the **damage
reduction formula**, and the reason this file is more than a table of numbers is that
there are *three* such formulas — one per shipped game — and the engine must pick the
right one from the shape of the data rather than from a version flag, because a helmet
section carries no version.

Its second job is incidental but visible: the helmet gates night vision, because the
goggles are part of the headgear.

## State

```text
RECORD Helmet                       # extends the base inventory item
  hit_type_protection[n]  : list<real>   # one factor per damage type; default 1
  bone_protection         : BoneProtections   # per-bone coefficient, armour value and
                                              # a pass-through flag; see BoneProtections
  bones_protection_sect   : text          # the section the per-bone table is read from
  night_vision_sect       : text          # empty means the helmet grants no night vision
  power_loss              : real          # invariant: clamped to 0..1
  health/radiation/satiety/power/bleeding restore speeds : real   # added to the wearer's rates
  nearest_enemies_show_dist : real        # radius at which the helmet marks enemies on the map
  uses_condition          : bool          # default true
```

**Invariants** — the protection list is sized to the full damage-type enumeration and
pre-filled with one, so an unlisted type is a pass-through rather than an index error.
Light burn is *always* aliased to burn and is never read separately; physical impact
defaults to the ordinary impact value. Fire wounds — bullets — default to zero, because
the bullet formula uses the per-bone armour value instead.

## `Load`

**Contract** — reads the section. Eight protection values are required; fire-wound and
physical-impact protection are optional, and the restore speeds, night-vision section,
per-bone section and enemy-marking distance all default to inert values.

**The version detection lives here and it is a heuristic.** A key naming the actor's hit
fraction exists in the data of two of the three games; among those two, the presence of a
fire-wound protection key distinguishes the older from the newer. The result selects one of
three damage formulas for the rest of the item's life.

**Notes** — the heuristic is explicitly fragile and the source says so: a modification that
adds a fire-wound key to newer data silently switches that helmet to the older formula. A
rebuild should record the game generation once, at startup — the engine already determines
it (see [`defines.h`](../xrEngine/defines.h.md)) — and pass it in rather than re-deriving
it per item. Reproducing the heuristic is only necessary to load unmodified data
identically.

## `HitThroughArmor`

**Contract** — the reason this class exists. Given the incoming damage, the bone that was
hit, the projectile's armour-piercing value and the damage type, returns the damage that
gets *through*, and may clear the caller's "add a wound" flag. Also wears the helmet.

Three formulas, selected by the detected generation. All three share one shape: **bullets
are resolved against per-bone armour with a penetration test; everything else is a flat
subtraction.**

```text
# --- newer generation ---
IF bullet THEN
  armor = bone_armor(bone) * condition
  IF bone_armor is negative THEN RETURN damage unchanged    # this bone is not covered
  IF piercing > armor THEN                                   # penetrated
    IF multiplayer THEN
      fraction = max((piercing - armor) / piercing, min_fraction)
      damage *= fraction * bone_coefficient
    # in single player a penetrating hit is NOT reduced at all
  ELSE                                                       # stopped
    damage *= min_fraction
    clear the caller's add-a-wound flag
ELSE
  scale = 1 for impact, wound, secondary wound and explosion; 0.1 otherwise
  damage -= default_protection(type) * scale, floored at zero
wear the helmet by the ORIGINAL damage

# --- older generation ---
IF bullet THEN
  armor = bone_armor(bone) * condition
  IF piercing > armor AND piercing > 0 THEN
    damage *= (piercing - armor) / piercing, floored at min_fraction
    IF multiplayer THEN damage *= bone_coefficient
  ELSE
    damage *= min_fraction; clear the add-a-wound flag
ELSE
  scale = 1 for wound, secondary wound and explosion; 0.1 otherwise
                                    # note: impact is NOT in this list here
  damage -= per_bone_protection(type, bone) * scale, floored at zero
wear the helmet by the RESULTING damage

# --- oldest generation ---
IF bullet THEN
  damage -= bone_armor(bone) * condition * (1 - piercing)
  floor the result at the original damage times min_fraction
ELSE
  damage -= per_bone_protection(type, bone)      # no floor at all
wear the helmet by the ORIGINAL damage
```

**Invariants** — the "stopped by armour" branch always clears the wound flag, in every
generation. A bullet that does not penetrate bruises rather than bleeds, and that is what
makes the minimum fraction meaningful: some damage always gets through, but it is not a
wound.

**Notes**

- The three differences that actually change gameplay are: whether the *impact* damage type
  gets the full or the tenth-strength subtraction; whether the non-bullet subtraction uses
  the flat protection or the per-bone one; and whether the helmet wears by the incoming or
  the reduced damage. None of these is a refinement of the others; they are three
  separately tuned games and the data was balanced against each.
- In the two newer generations the per-bone coefficient is applied to a penetrating bullet
  **only in multiplayer**. In single player a bullet that beats the armour does full
  damage. That asymmetry is deliberate balance, not a bug.
- A negative bone-armour value means "not covered by this helmet" and short-circuits the
  whole bullet path in the newest formula only.

## `GetDefHitTypeProtection` · `GetHitTypeProtection` · `GetBoneArmor`

**Contract** — the protection factor for a damage type, with and without the per-bone
coefficient, scaled by the helmet's condition so that a worn helmet protects less. The
oldest generation stores these as *pass-through* fractions and the two newer ones as
*blocked* fractions, so the oldest returns one minus the value. That inversion is a data
convention, not a computation.

## `ReloadBonesProtection` · `AddBonesProtection`

**Contract** — builds the per-bone table against the *wearer's skeleton*, because the table
is keyed by bone name and the names must be resolved to indices. The wearer is the item's
parent, except in single player, where it is always the currently viewed entity — the
player — regardless of who actually owns the helmet. Adding merges a second section's
values on top of the first rather than replacing them, which is how an upgrade adds armour
to specific bones.

**Notes** — using the viewed entity rather than the parent in single player is a
simplification that holds only because in single player the only creature whose helmet
protection is computed this way is the player. A rebuild should resolve against the actual
wearer.

## `net_Spawn` · `net_Export` · `net_Import`

**Contract** — the per-bone table is built at spawn, before the base class spawns the
object, and only in single player. On the wire the helmet contributes exactly one value:
its condition, quantized to a single byte over the unit range. Nothing else about a helmet
changes at run time.

**Invariants** — the quantization is part of the frozen protocol between this codebase's
own client and server; a rebuild may change it only if it changes both ends.

## `OnMoveToSlot` · `OnMoveToRuck`

**Contract** — putting a helmet on re-enables night vision if the torch item was already
switched to night vision; taking it off switches night vision off. Only transitions *from a
slot* are considered, so moving a spare helmet around the rucksack does nothing.

**Notes** — the coupling runs through the torch item because the goggles and the headlamp
are one item in this game's model. A rebuild is free to separate them, at the cost of
diverging from the shipped data's slot layout.

## `Hit`

**Contract** — wears the helmet. The incoming damage is scaled by the item's own immunity
for that damage type before being subtracted from its condition — note this is the *item's*
immunity, not the protection values above, so a helmet's rate of wear and its protection
are tuned independently.

## `install_upgrade_impl`

**Contract** — applies an upgrade section: nine protection values, the five restore speeds,
the stamina-loss factor, the enemy-marking distance, the night-vision section, and both a
*replacement* and an *additive* per-bone section. The additive form is the interesting one —
it is how an upgrade adds plating over specific bones without restating the whole table.
The actor's hit fraction is upgradeable only in the two newer generations, because the
oldest stores it elsewhere.
