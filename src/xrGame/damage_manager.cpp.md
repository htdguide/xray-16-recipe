# src/xrGame/damage_manager.cpp

> Per-bone damage multipliers: how much a hit on this bone hurts, how much it wounds, and whether the first aimed shot gets its own number.

**Needs** — [`damage_manager.h`](damage_manager.h.md) · [`xrEngine/xr_object.h`](../xrEngine/xr_object.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: two scaling factors read from a skeleton; the only awkwardness is where they are stored

## Purpose

A headshot must matter more than a hit on the shin, and the ratio must be authored per
creature, not hardcoded. This mix-in reads that table from configuration once at load and
answers the one question the hit pipeline asks: given the bone that was struck, what does
the damage and the wound get multiplied by.

It is a mix-in on the object rather than a table beside it because the numbers are stored
**in the skeleton's bone instances**, one slot per bone, alongside the animation transform.
That is the single most surprising decision in the file and it is discussed below.

## State

```text
RECORD DamageManager
  default_hit_factor   : real    # multiplier for a bone with no entry, and for "no bone"
  default_wound_factor : real
  object               : GameObject   # the carrier, resolved at construction
```

The per-bone numbers are **not** here. They live in the carrier's skeleton, four parameter
slots per bone instance:

```text
slot 0 : hit scale          # damage multiplier
slot 1 : bone hit type      # which damage channel this bone belongs to, stored as a real
slot 2 : wound scale        # bleeding-wound multiplier
slot 3 : first-bullet hit scale   # see the aimed-shot rule below
```

Invariant: **every bone of the carrier's skeleton has all four slots written** before any
hit can be resolved, because the lookup does no bounds or presence check. The load path
guarantees it by writing defaults to every bone first and only then overlaying the authored
entries.

**Notes** — storing gameplay multipliers in animation bone instances is a space
optimisation: every creature already has one bone instance per bone, the table is exactly
per-bone, and this avoids a parallel array and a second lookup in the hit path. The cost is
that the damage table is silently coupled to the model, shared with the animation system,
and invisible to anyone reading the damage manager's own fields. A rebuild should keep the
*shape* — a dense per-bone record on the instance, reachable in one indexed step from the
bone index the collision result gives you — and is free to make it a separate array rather
than four untyped slots on the pose.

## `reload`

**Contract** — reads the damage table for a configuration section. Two spellings: one takes
the section that *is* the table; the other takes a section and a key within it whose value
names the table section, falling back to the first spelling with no configuration at all
when the key is absent. Requires the carrier's visual to exist — it writes into the
skeleton. Does not allocate.

```text
FUNCTION reload(section, config)
  default_hit_factor   = 1
  default_wound_factor = 1
  present = config exists AND config has section

  IF present AND section has a "default" line
    # the line is a comma-separated triple; only fields 0 and 2 are read
    default_hit_factor   = real(field 0 of the "default" line)
    default_wound_factor = real(field 2 of the "default" line)

  init_bones(section, config)        # write the defaults to EVERY bone
  IF present
    load_section(section, config)    # overlay the authored per-bone entries
```

**Invariants** — the order is load-bearing and is the reason the defaults are parsed before
the bones are initialised: every bone must carry the section's own default, not a global
one, before any authored line overrides it. A rebuild that seeds bones with a fixed 1 and
then applies the default line only to unlisted bones will get the same result by a longer
route; doing it in this order means the overlay pass never has to know which bones it did
not touch.

The indirect spelling — a section naming another section — exists because several entity
sections share one damage table. A rebuild may resolve the indirection at configuration-load
time instead.

## `init_bones`

**Contract** — private. Writes the section's defaults into every bone of the carrier's
skeleton: the default hit factor, a hit type of one, and the default wound factor. Slot 3,
the first-bullet scale, is **not** initialised here.

**Notes** — that omission is load-bearing, not an oversight, and it is what makes the aimed
-shot rule work: an uninitialised or zero first-bullet scale means "this bone has no special
first-shot number", and the lookup falls back to the ordinary hit scale. A rebuild must
either reproduce "zero means unset" or carry an explicit optional.

The hit-type default of one is a magic number the file does not explain. It selects a damage
channel for the bone, and the channel names live with the hit types rather than here.

## `load_section`

**Contract** — private. Walks every line of the table section except the default one. Each
key is a bone name and each value is a comma-separated list: hit scale, hit type, wound
scale, and optionally a first-bullet hit scale. An unknown bone name is a hard failure — it
is authoring data and a silent miss would make a creature quietly invulnerable somewhere.

```text
FUNCTION load_section(section, config)
  FOR EACH (bone_name, value) IN config[section]
    IF bone_name == "default": CONTINUE
    bone = skeleton.bone_index(bone_name)
    IF bone is none: FAIL WITH bone_name
    slot0 = real(field 0 of value)
    slot1 = real(integer(field 1 of value))     # hit type, stored in a real slot
    slot2 = real(field 2 of value)
    IF value has fewer than 4 fields
      slot3 = slot0                             # no separate first-bullet number
    ELSE
      slot3 = real(field 3 of value)
    IF bone is the root AND (slot0 is zero OR slot2 is zero)
      FAIL WITH "hit_scale and wound_scale for root bone cannot be zero"
```

**Invariants** — the root bone's hit and wound scales must both be non-zero. The root is the
fallback target for any hit the collision system cannot attribute to a specific bone, so a
zero there makes the creature invulnerable to a whole class of hits, and the symptom
(a creature that ignores explosions but dies to bullets) is very hard to trace back to one
configuration line. Hence the explicit check, with the offending section named.

**Notes** — the three-field form copies the ordinary hit scale into the first-bullet slot,
which means "the first aimed shot is no different here". The four-field form is how a
creature is given a distinct first-shot vulnerability. Note the asymmetry with
`init_bones`, which leaves the slot at zero: a bone with *no line at all* has an unset
first-bullet scale and falls back at lookup time, while a bone with a three-field line has
an explicitly equal one. The two paths agree in result and disagree in mechanism.

## `HitScale`

**Contract** — the query the hit pipeline makes. Takes the struck bone and a flag saying
whether this is an aimed first bullet, and yields the hit multiplier and the wound
multiplier. Pure; no allocation; called once per hit.

```text
FUNCTION hit_scale(bone, aim_bullet) -> (hit_scale, wound_scale)
  IF bone is none                                   # the hit was not attributed to a bone
    RETURN (default_hit_factor, default_wound_factor)

  scale = 0
  IF aim_bullet
    scale = skeleton.bone(bone).slot3               # first-bullet scale
  IF NOT aim_bullet OR scale is zero
    scale = skeleton.bone(bone).slot0               # ordinary scale
  RETURN (scale, skeleton.bone(bone).slot2)
```

**Notes** — two rules are encoded in those four lines.

*The unattributed hit uses the section defaults.* When the collision system reports no bone —
an explosion, a scripted hit — the multiplier is the creature-wide default rather than one.
So a creature authored as generally tough is tough against explosions too.

*The first-bullet scale is a first-class override with a fallback.* The aimed-shot path
reads slot 3 and, if it is zero (never authored), silently uses the ordinary scale. This is
the mechanism behind the "first shot from concealment does extra damage" behaviour tuned per
bone; see [`first_bullet_controller.cpp`](first_bullet_controller.cpp.md) for who decides a
shot qualifies.

*The wound multiplier has no first-bullet variant.* Bleeding is scaled the same way whether
or not the shot was aimed. Nothing in the source suggests that was considered and rejected.

## `_construct`

**Contract** — resolves and stores the carrier, which the mix-in cannot know at construction
because it is not itself the object. Called during the object's own construction, before any
configuration is read.

**Notes** — incidental to the original's two-phase construction of game objects. A rebuild
that can pass the carrier to the constructor needs none of it; what survives is the
ordering requirement that the carrier be known before `reload` runs, since `reload` writes
into the carrier's skeleton.
