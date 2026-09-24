# src/xrGame/ef_primary.cpp

> The leaf evaluation functions: each reads one property off the creature or item currently in the shared parameter block, from whichever of the two worlds — live objects or alife records — is in use.

**Needs** — [`ef_primary.h`](ef_primary.h.md) · [`ef_storage.h`](ef_storage.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`Weapon.h`](Weapon.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_human_brain.h`](../xrServerEntities/alife_human_brain.h.md) · [`alife_human_object_handler.h`](alife_human_object_handler.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: property reads with a two-world dispatch

## Purpose

Thirty small functions, each answering one question about one of the four slots of the
[shared parameter block](ef_storage.h.md). Individually they are trivial; the file's content
is the *pattern* they all follow and the handful that deviate from it.

## State

Stateless, except that several functions **rewrite their own declared maximum** as a side
effect of being evaluated. Health does it from the creature's own maximum health.

## The universal shape

Every function in the file is this:

```text
FUNCTION value()
  IF the client-side parameter block has a member THEN
    read the property off the live object
  ELSE
    read the property off the alife record
```

**Invariants** — the test is always on the *member* slot of the client block, even in
functions that only read an item or an enemy. Emptiness of that one slot is the system's
mode flag; see [`ef_storage_inline.h`](ef_storage_inline.h.md).

**Invariants** — the alife branch downcasts the record to the abstract monster or human
record and treats a failure as a contract violation, not a miss. Asking a non-creature for
its morale is a bug in the caller.

**Invariants** — many functions assert that the value they are about to return is within the
declared range. Those are the categorical ones, where a value outside the range would index
a trained table out of bounds.

## The functions that read a plain property

**Contract** — `PersonalHealth`, `PersonalMorale`, `PersonalAccuracy`,
`PersonalIntelligence`, `PersonalEyeRange`, `PersonalCreatureType`, `EquipmentType`,
`MainWeaponType`, `EnemyAnomalyType`, `ItemValue`, `GraphPointType0` each read the named
field from whichever world is live. Nothing more.

**Notes** — `PersonalHealth` additionally sets its own maximum to the subject's maximum
health before returning, so the discretization is a *fraction of that creature's own health*
rather than of an absolute scale. That is the right model and it is the reason the declared
maximum in the header is only a placeholder.

**Notes** — `PersonalEyeRange` and `PersonalMaxHealth` have **no client-side branch at all**:
they unconditionally read an alife record and fail if there is not one. They are alife-only
functions that never got their other half.

## `Distance`

**Contract** — the straight-line distance between the member and the enemy, from live
positions or record positions. It does not test whether the enemy slot is filled; evaluating
distance with no enemy is a crash rather than a reported error.

## `PersonalWeaponType` · `ffGetTheBestWeapon`

**Contract** — the type of the weapon the creature would fight with. Three cases, and the
third is the interesting one:

```text
FUNCTION value()
  IF the creature has a natural weapon THEN RETURN its own weapon type   # claws, teeth
  RETURN best_weapon()

FUNCTION best_weapon()
  IF an item slot is already filled THEN RETURN that item's weapon type   # asked about a
                                                                          #   specific weapon
  IF this is the client-side world THEN
    best = 0
    FOR EACH inventory slot of the creature
      IF the item is a weapon AND its total suitable ammunition exceeds
         a tenth of its magazine size THEN
        temporarily install it as the member item
        best = max(best, its weapon type)
        clear the member item again
    RETURN best
  ELSE
    IF the record has no recorded best weapon THEN RETURN 0
    install that weapon as the member item ; RETURN its weapon type
```

**Invariants** — a weapon with less than a tenth of a magazine's worth of ammunition is not
considered. A creature carrying an empty rifle evaluates as unarmed, which is what makes the
alife simulation give it a fight it can survive.

**Invariants** — the "best" weapon is the one with the highest *type number*, so the weapon
type enumeration is ordered by lethality. That ordering is a frozen property of the
enumeration and of the trained tables that consume it.

**Invariants** — the client-side search mutates the shared parameter block's item slot and
restores it to *empty*, not to its previous value. A caller that had filled the item slot
before asking loses it. The alife branch leaves its installed weapon in place permanently.
Both are the block's shared-state hazard showing through; a rebuild passing arguments has
neither.

## `PersonalMaxHealth`

**Contract** — the creature's maximum health, multiplied by the group size when the subject
is a *group* record rather than an individual. An offline squad is one record with a count,
and its effective health is the sum.

**Its discretization is hand-written and non-linear**:

```text
FUNCTION discrete(n)
  v = value()
  clamp to bucket 0 below the minimum and bucket n-1 above the maximum
  bucket = round(k * n / 10) where k is chosen by:
    v <= 30 → 1 ;  v <= 50 → 2 ;  v <= 80 → 3 ;  v <= 100 → 4 ;  v <= 150 → 5
    v <= 250 → 6 ; v <= 500 → 7 ; v <= 750 → 8 ; otherwise 9
```

**Invariants** — the break points are roughly geometric, tight at the low end and coarse at
the high end, because the difference between thirty and fifty health decides a fight and the
difference between five hundred and seven hundred and fifty does not. The buckets are then
expressed as tenths of the requested resolution, so the same break points work for any table
width.

**Invariants** — note the scaling uses `n`, not `n - 1`, unlike the base rule. The top
category maps to nine tenths of `n`, which for a ten-bucket table is bucket nine — the last —
and for smaller tables is not. A rebuild must reproduce the arithmetic exactly, not the
intent.

## `WeaponAmmoCount`

**Contract** — how much ammunition the creature has for a given weapon, computed by the alife
human's object handler across its whole inventory. Client-side, unimplemented, returns zero.

**Its discretization is also hand-written**, and it reaches back into the game data:

```text
FUNCTION discrete(n)
  v = value()
  clamp at both ends as usual
  IF the item is a weapon record with an ammunition section THEN
    box = the "box_size" of its FIRST listed ammunition type
    RETURN round( (v <= 3*box ? 1 : 2) * n / 10 )
  RETURN n - 1        # unknown ammunition: assume plenty
```

**Invariants** — the threshold is *three boxes* of the weapon's own ammunition, so "enough"
is relative to the calibre rather than absolute. A rebuild must read the box size from
configuration; hard-coding a round count breaks for every weapon whose boxes differ.

**Invariants** — the fallback when the ammunition type is unknown is the *top* bucket, not
the bottom. An unidentifiable weapon is treated as fully supplied, which biases the alife
simulation toward letting fights happen.

## `ItemDeterioration`

**Contract** — how worn an item is, as one minus its condition. Client-side it handles only
weapons and returns zero for anything else; the alife branch reads the record's stored
deterioration for any inventory item.

## `EquipmentPreference` · `MainWeaponPreference`

**Contract** — how much *this particular creature* wants this kind of equipment, looked up in
a per-creature preference array indexed by the discretized equipment or weapon type. Alife
only; the client-side branch returns zero.

**Invariants** — the preference array is indexed by a *discretized* type, and the bucket
count passed is the type function's own declared maximum. So the preference array's length is
tied to the type enumeration's size, in two places that must agree.

**Notes** — the file carries two complete copies of these three functions behind a build
switch, differing only in whether the preferences are reached through the alife human's
*brain* object or directly off the record. The brain-mediated version is the one built. The
switch is a refactoring left half-finished; a rebuild keeps one.

## `DetectorType`

**Contract** — what kind of anomaly detector the creature effectively has: its own, if it
detects anomalies naturally, otherwise the carried item's. Returns zero when no item is in
the slot at all, which is the only function in the file with a graceful empty-slot answer.

## `EnemyDistanceToGraphPoint`

**Contract** — already a bucket, not a distance: zero below five metres, then one, two, three
at five-metre steps, and four beyond twenty. Alife only. The break points exist because the
trained tables were fitted with them.

## `EnemyRukzakWeight`

**Contract** — the total weight the subject is carrying, client-side only. The alife branch
is commented out and returns zero, so the alife simulation cannot see what a creature is
carrying, which is the input this function exists to supply.

## The unimplemented functions

**Contract** — `PersonalRelation`, `PersonalGreed`, `PersonalAggressiveness`,
`EnemyEquipmentCost` and `EnemyAnomality` return a constant zero. Each carries a note naming
what it was meant to compute.

**Invariants** — they are still registered in the catalogue at their slots and still feed any
trained table that lists them as an input. A constant input contributes a constant weight, so
those tables are effectively fitted with a dead feature. This is worth stating plainly: five
of the thirty leaf functions are stubs, and a rebuild that implements them properly will not
reproduce the original's behaviour, because the trained tables were fitted against the stubs.

## `clsid_member` · `clsid_enemy` · `clsid_member_item` · `clsid_enemy_item`

**Contract** — the class identifier of each parameter slot, from the live object's tag or the
record's stored one. The four are declared in [`ef_base.h`](ef_base.h.md) and defined here so
they can see both worlds' types. Asking for an empty slot is a contract violation.
