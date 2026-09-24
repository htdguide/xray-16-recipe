# src/xrGame/BoneProtections.cpp

> The per-bone armour table an outfit or helmet is described by: for each named bone, how much damage it deflects, how much penetration it stops, and whether bullets pass through it at all.

**Needs** — [`BoneProtections.h`](BoneProtections.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`game_type.h`](game_type.h.md)
**Used by** — [`BoneProtections.h`](BoneProtections.h.md)
**Tier floor** — T3: parses a configuration section into a lookup table

## Purpose

Damage in this game is located: a hit names the bone it landed on, and armour is authored
per bone. This file turns a configuration section whose keys are *bone names* into a table
keyed by *bone index*, resolved against the wearer's skeleton, and answers three queries
against it with a defined fallback.

It is shared by every kind of armour — outfits, helmets, and the per-creature armour of
non-player characters — which is why it is a free-standing record rather than a member of
one of them.

## State

```text
RECORD BoneProtection
  coefficient      : real = 1.0   # multiplies damage that gets through
  armor            : real = 0.0   # the penetration value a bullet must beat
  bone_passes_bullet : bool       # true means a bullet continues through this bone

RECORD BoneProtections
  by_bone      : map<int, BoneProtection>   # keyed by bone index; the invalid index
                                            # holds the DEFAULT entry, always present
  hit_fraction : real = 0.1       # the minimum share of damage that always gets through
  fraction_kind : enum { oldest, actor_newer, creature, actor_newest }
```

**Invariant** — the entry under the *invalid* bone index is the default and is guaranteed
to exist, which is what makes every lookup total: an unlisted bone falls back to it rather
than failing. That is the whole reason the invalid index is used as a map key rather than
as a sentinel.

**Invariant** — a bone name in the data that the skeleton does not have is a hard failure,
not a skipped line. Armour authored for a bone that does not exist means the section was
written for a different model, and silently ignoring it would produce an unarmoured wearer
with no diagnostic.

## `reload`

**Contract** — clears the table and rebuilds it from one section against one skeleton.
Every key that is not one of the two hit-fraction keys is a bone: its value is a
comma-separated triple of coefficient, armour and a pass-through flag. The key `default`
fills the fallback entry instead of naming a bone.

**The fraction kind is decided here, and only when the caller has not already decided it.**
Outfits and helmets set it themselves from their own data shape (see
[`ActorHelmet.cpp`](ActorHelmet.cpp.md)); for everything else — creatures — the section is
asked, preferring the creature-specific key over the generic one, and falling back to the
*game-material library's version number* when neither key is present. That last fallback is
the engine's only version signal available at this point.

```text
FUNCTION reload(section, skeleton)
  IF the caller has not already fixed the fraction kind THEN
    IF the section has the creature hit-fraction key THEN kind = creature; read it
    ELSE IF it has the generic key                   THEN kind = oldest;  read it
    ELSE kind = (material library is the newest generation) ? creature : oldest
         hit_fraction = 0.1

  clear the table; install the default entry with its built-in defaults
  FOR EACH (key, value) IN the section
    SKIP the two hit-fraction keys
    entry = { coefficient: value[0], armor: value[1], passes_bullet: value[2] > 0.5 }
    IF key is "default" THEN replace the fallback entry
    ELSE
      index = skeleton.bone_index(key)
      FAIL WITH "no such bone" IF the index is invalid
      insert entry at index
```

**Notes**

- The pass-through flag is stored as a number in the data and compared against a half. A
  rebuild should read it as a boolean; the half is an artifact of every value in the tuple
  being parsed as a number.
- Values are parsed positionally out of a comma-separated list. A section that omits the
  third field yields a zero, and therefore a bullet-stopping bone. That default matters.
- The insert for a named bone does **not** overwrite an existing entry, while the default
  entry does. So a section listing the same bone twice keeps the first; a section listing
  `default` twice keeps the last. That inconsistency is real; no shipped section does
  either.

## `add`

**Contract** — merges a second section's values *additively* onto an existing table: the
coefficient and armour of each named bone are increased, and the hit fraction is increased
by the matching key if present. This is how an upgrade adds plating without restating the
table. Unlike a reload, a bone not already in the table is created with the addend as its
value, because the map insert default-constructs.

**Invariants** — the pass-through flag is **not** merged; an additive section cannot change
whether a bone stops bullets. Merging is a no-op outside single player, because upgrades
are a single-player mechanic.

**Notes** — because the addend is applied to a default-constructed entry when the bone is
new, adding a coefficient of 0.2 to an unlisted bone yields 1.2, not 0.2. That is almost
certainly not what an author writing an additive section expects, and is worth flagging in
a rebuild rather than reproducing blindly — though the shipped data only ever adds to bones
the base section already lists.

## `getBoneProtection` · `getBoneArmor` · `getBonePassBullet`

**Contract** — the three lookups, each falling back to the default entry when the bone is
not in the table. All three are total; none can fail.
