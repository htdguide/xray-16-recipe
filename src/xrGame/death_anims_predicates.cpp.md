# src/xrGame/death_anims_predicates.cpp

> The seven conditions that decide which authored death animation a fatal hit earns, and the geometry that turns a hit direction into one of four body-relative quadrants.

**Needs** — [`death_anims.h`](death_anims.h.md) · [`Actor.h`](Actor.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`WeaponShotgun.h`](WeaponShotgun.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Explosive.h`](Explosive.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`animation_utils.h`](animation_utils.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: geometry and type tests over the hit record

## Purpose

[`death_anims.cpp`](death_anims.cpp.md) owns the table; this file owns the questions it
asks. Each of the seven kill types is a predicate over the fatal hit record: who fired,
with what, from how far, into which bone, with what momentum. The first predicate to say
yes and yield a valid motion wins.

One decision dominates the file and belongs at the top: **every predicate first requires
that the killer is the entity the player is currently controlling.** Scripted death
animations are a spectacle for the player. A stalker shot by another stalker off to the
side always falls as a ragdoll, which costs nothing because nobody was framing the shot.

## State

Stateless — the predicate objects hold only the motion bags the table gave them.

## `type_motion::dir`

**Contract** — classifies the hit into one of four body-relative quadrants and writes out
the residual angle within that quadrant. Total: a hit with no horizontal direction at all
(straight down, typically an explosion overhead) is assigned a **uniformly random**
quadrant rather than failing, leaving the angle untouched.

```text
FUNCTION dir(entity, hit) -> (direction, angle)
  d = hit.direction with the vertical component removed
  IF d is zero THEN RETURN (random one of the four, angle unchanged)
  d = normalize(d)
  forward = normalize(entity forward axis, flattened)
  right   = normalize(entity right axis,   flattened)
  f = dot(d, forward)        # how much the shot travels along the body's facing
  s = dot(d, right)

  IF |f| > sqrt(1/2) THEN                       # within 45 degrees of the body axis
    sign = (f < 0) ? -1 : +1
    angle = atan2(-sign*s, sign*f)
    RETURN (sign < 0 ? front : back, angle)     # see the inversion note
  ELSE
    sign = (s > 0) ? +1 : -1
    angle = atan2(sign*f, sign*s)
    RETURN (sign > 0 ? left : right, angle)
```

**Invariants** — the quadrant boundaries are at exactly forty-five degrees, which is what
the square root of one half encodes; the four quadrants tile the circle with no gap and no
overlap.

**Invariants** — the direction is the direction the projectile *travels*, not the direction
it came from. A shot travelling along the body's facing struck it in the **back**, which is
why the sign test reads inverted. Getting this backwards swaps every front and back
animation in the game and is the single easiest mistake to make here.

**Invariants** — the angle is measured from the quadrant's own axis, so it is always within
forty-five degrees of zero for the front/back case, and the caller adds it to the body's
yaw to align an animation authored for a dead-on hit with the shot that actually landed.

## `global_hit_position`

**Contract** — converts the hit's bone-local impact point into world space, through the
bone's current pose and then the entity's transform. Requires a visual with a skeleton.
Used by the two distance-gated predicates, which need the distance to the *wound*, not to
the entity's origin — at shotgun ranges the difference between a chest and a foot is a
significant fraction of the threshold.

## `is_bone_head`

**Contract** — true when the struck bone is the neck, or is the head bone or anything
parented beneath it. Bone identity is by authored name (`bip01_head`, `bip01_neck`), which
freezes those two names across every humanoid model the game ships. A model naming them
otherwise gets no headshot animations and no diagnostic.

## `is_snipper`

**Contract** — true when the weapon that fired is a magazined weapon that is **both**
currently scoped-in and has a scope attached. It does not test calibre or the weapon's
class: "sniper" here means *the player was aiming down a scope at the moment of the kill*,
which is the cinematic condition, not the ballistic one. A single-shot-mode requirement was
considered and is commented out.

## The seven predicates

Listed in the order [`death_anims.cpp`](death_anims.cpp.md) evaluates them, which is not the
order they are declared.

### 0 — running headshot

**Contract** — the momentum case: a moving target shot in the head keeps going forward. All
of the following must hold, and each one is a separate reason to reject:

- the struck bone is in the head group;
- the creature has character physics that is not the *biting* kind — that excludes animals,
  whose deaths are handled elsewhere;
- its speed is at least **3.65** metres per second, which is the shipped run speed: the
  animation only reads as momentum if the body was actually running;
- its velocity points within **20 degrees** of the direction to the killer — it was running
  *at* the player, not across;
- the classified quadrant is **front** (the shot came head-on);
- the killer is within **30** metres of the wound.

**Notes** — the speed threshold, the cone and the range are all authored into the code
rather than the configuration, so this case cannot be retuned without a rebuild. The 3.65
figure is not derived from anything visible; it matches the stalker run speed in the shipped
data and should be read as "at a run", not as a physical constant.

### 1 — riddled by a burst

**Contract** — **never matches.** The type exists, its configuration key `kill_burst` is
parsed and its motion bags are loaded, and the predicate returns no unconditionally. The
feature was designed and not finished. A rebuild should either implement it — the missing
piece is a count of hits from one weapon within a short window, which nothing in the hit
record carries — or drop the slot and the key together.

### 2 — shotgun

**Contract** — the killing weapon is a shotgun and the killer is within **20** metres of the
wound. Direction is classified normally. The range gate is what distinguishes the authored
"blown backwards" set from an ordinary long-range kill with the same weapon.

### 3 — grenade

**Contract** — matches on either of two signals: the hit's damage type is *explosion*, or
the object that caused it is an explosive. The second branch catches **fragments**, which
arrive as ordinary damage from a fragment object rather than as explosion damage. Both
branches classify direction normally.

### 4 — sniper, head

**Contract** — scoped weapon and a head-group bone.

### 5 — sniper, body

**Contract** — scoped weapon and *not* a head-group bone. Deliberately the complement of the
previous one, so the two together cover every scoped kill and the general headshot case
below never sees one.

### 6 — headshot

**Contract** — any remaining head-group hit. The catch-all that the three more specific
head cases get first refusal on, which is why it occupies the last slot.

## `type_motion_diagnostic`

**Contract** — debug-build-only logging of one selection: which predicate fired, which
quadrant, which motion by name, which creature, which visual, which bone. Silent unless the
debug switch is on. It is the only observability the selector has, and the reason the
direction-name token table in [`death_anims.cpp`](death_anims.cpp.md) still exists.
