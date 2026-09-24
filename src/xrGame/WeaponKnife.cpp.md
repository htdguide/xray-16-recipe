# src/xrGame/WeaponKnife.cpp

> The knife: two distinct attacks, each landing on an animation marker, each resolved as a small burst of short-range hits aimed at the bone shapes inside a sphere in front of the player.

**Needs** — [`WeaponKnife.h`](WeaponKnife.h.md) · [`Weapon.h`](Weapon.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`Actor.h`](Actor.h.md) · [`Level.h`](Level.h.md) · [`player_hud.h`](player_hud.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../xrCore/Animation/SkeletonMotions.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a spatial query and a per-bone shape intersection pass per swing.

## Purpose

A melee weapon has to answer a question a firearm never asks: *what did that swing hit?*
A bullet is a ray; a knife sweep is a volume, and the thing it sweeps through is a
skeleton, not a bounding box. So the knife's whole substance is a targeting search — find
the bone shapes inside a sphere in front of the player, pick the best few, and fire a
very short-range, knife-material hit at each.

Two attacks exist and differ in more than damage: a **slash** (the primary) spreads
across several victims, and a **thrust** (the secondary) concentrates on one. That
difference is expressed entirely in the search's parameters.

The knife is not a magazined weapon. It inherits the weapon base directly and runs its
own four-state machine.

## State

```text
RECORD Knife EXTENDS Weapon

  # --- per-attack authored parameters, one set per attack -------------
  hit_type[2]        : HitType        # burn, wound, strike, ... from the damage model
  hit_power[2][difficulty] : real
  hit_power_critical[2][difficulty] : real
  hit_impulse[2]     : real
  reach[2]           : real           # how far in front the search sphere's centre sits
  splash_direction[2] : vector3       # which way the sweep leans, in camera space
  splash_radius[2]    : real          # the search sphere's radius
  hits_count[2]       : int           # how many hits the attack may produce
  per_victim_count[2] : int           # how many of those may land on one creature

  next_hit_divide_factor : real = 0.75  # each successive hit of a swing is weaker

  # --- the attack currently resolving --------------------------------
  current_hit_type   : HitType
  current_hit_power, current_hit_impulse : real
  reach, splash_direction, splash_radius, hits_count, per_victim_count : the chosen set

  old_strike_method  : bool           # true when the section carries no splash parameters
  attack_started     : bool
  attack_has_motion_marks : bool
  wallmark_size      : real
  knife_material     : int (16-bit)
```

**Invariants** —
- the ten splash parameters are **all or nothing**: a section supplying some but not all
  is a hard error. Supplying none selects the old single-ray strike;
- `current_*` are set on entering a fire state and read by everything downstream, so the
  attack's identity is carried in mutable fields rather than passed. A rebuild should
  pass the chosen parameter set explicitly;
- the primary attack allows several victims but caps hits per victim; the secondary
  allows one victim and no per-victim cap. That inversion is the slash/thrust
  distinction.

## `Load` — the all-or-nothing table

**Contract** — reads the wallmark size, the strike sound and the knife's own material
identity, then walks a ten-entry table of splash parameters, each with a name and an
optional fallback name.

**Notes** — four of the ten carry fallback names that are *misspellings* (`spash1_dist`
for `splash1_dist`, and three more) shipped in the original data and never corrected.
Both spellings must be accepted.

The table's entries for `splash1_radius` and `splash2_radius` both write the **first**
radius. The second attack's authored radius is therefore never read, and a thrust uses
the slash's radius. This is a bug in the current source, not a historical decision, and a
rebuild should fix it — but note it changes the thrust's reach.

The strictness is the interesting decision: a section with a partial parameter set fails
loudly rather than filling in defaults, listing exactly which names were missing. Melee
reach is felt, not read, so a silently wrong default would be discovered by players
rather than by authors.

## The four-state machine

**Contract** — the knife's states are showing, idle, hiding, hidden, and two firing
states (primary and secondary). Entering either firing state selects that attack's
parameter set — including reading `hit_power` at the *current difficulty* for an
actor-held knife in single player, and at the hardest setting otherwise — and starts its
animation.

```text
FUNCTION switch_to_attacking(knife, state)
  RETURN IF the knife is pending                       # one swing at a time
  play the attack animation for this state, without blending
  knife.attack_has_motion_marks = the played motion carries markers
  knife.attack_started = true
  knife.pending = true
```

**Invariants** — a knife-carrying stalker always hits at the hardest difficulty's damage,
and so does an actor in multiplayer. Only a single-player actor's knife scales with the
difficulty setting.

## When the blow lands

**Contract** — the hit is timed by the *animation*, in one of two ways depending on what
the animation data provides.

```text
# preferred: the animation carries a marker at the frame the blade connects
ON motion marker DURING an attack state:
  resolve the strike

# fallback: the animation has no markers
ON attack animation END:
  IF the attack had started THEN
    clear attack_started
    play the attack's follow-through animation
    IF a follow-through exists AND the animation had no markers THEN
      resolve the strike                    # land it here instead
  IF no follow-through exists THEN
    return to idle
```

**Invariants** — this is the file's most load-bearing coupling. The animator decides when
a knife connects by placing a marker; the fallback exists only for animation sets that
predate markers, and lands the blow at the *end* of the swing instead, which feels
noticeably later. An attack animation is followed by a separate follow-through animation
(the recovery), and only when that is absent does the weapon return straight to idle.

## `OnKnifeStrike` — arming the attack

**Contract** — selects the current attack's parameter set into the `current_*` fields,
derives the weapon's effective range as reach plus splash radius, asks the carrier for
its aim, and resolves the strike from there.

```text
FUNCTION on_knife_strike(knife, state)
  MATCH state
    primary   -> current set = attack 1's reach, direction, radius,
                 hits_count, per_victim_count
    secondary -> current set = attack 2's, with per_victim_count = 0
    otherwise -> RETURN
  knife.fire_distance = reach + splash_radius
  RETURN IF there is no carrier
  ask the carrier for its aim: position, direction
  strike(knife, position, direction)
```

## `KnifeStrike` — the three-way resolution

**Contract** — the top of the targeting search. Three outcomes, tried in order.

```text
FUNCTION strike(knife, position, direction)
  IF knife.old_strike_method THEN
    make_shot(knife, position, direction, 1.0)      # one hit straight ahead
    RETURN

  # 1. something is directly in front within reach: hit it, hard
  victim = ray pick along direction, up to reach, ignoring the carrier
  IF victim exists THEN
    strength = per_victim_count for a slash, hits_count for a thrust
    make_shot(knife, position, direction, strength)  # ONE hit worth several
    RETURN

  # 2. nothing directly ahead: search the sphere for bone shapes
  targets = select_hits(knife, position)
  IF targets is non-empty THEN
    strength = 1.0
    FOR EACH target IN targets
      make_shot(knife, position, normalize(target - position), strength)
      strength = strength * knife.next_hit_divide_factor    # 0.75 each time
    RETURN

  # 3. nothing found: swing at the air, which still marks walls
  make_shot(knife, position, direction, 1.0)
```

**Invariants** — the direct-pick shortcut collapses what would have been several hits
into one hit of several times the strength. That keeps the total damage against a target
squarely in front the same as against one found by the search, while spending one hit
instead of three — and it is why stabbing someone point-blank does not multiply damage.

The successive-hit falloff is geometric at 0.75, so a three-hit slash deals 1 + 0.75 +
0.5625 = 2.3125 times one hit's damage rather than three times. That is the entire reason
a multi-target slash is not strictly better than a thrust.

## `MakeShot` — a knife hit as a bullet

**Contract** — a knife hit is delivered through the **bullet manager**, as a projectile
with knife parameters: no tracer, no ricochet, negligible armour piercing, the knife's
own material identity, and the weapon's very short range.

```text
FUNCTION make_shot(knife, position, direction, strength)
  cartridge = a synthetic round:
      one projectile, no wear, no dispersion,
      hit factor = strength, impulse factor = 1,
      no tracer, no ricochet, armour piercing = negligible,
      wallmark size = knife.wallmark_size,
      material = the knife material
  ensure the magazine holds at least 2 rounds of it; ammo_elapsed = magazine length
  play the strike sound at the position
  IF the carrier is the actor THEN buzz the gamepad triggers, scaled by strength
  hand the bullet manager a projectile from position along direction with
      knife.current_hit_power, knife.current_hit_impulse, knife.current_hit_type,
      knife.fire_distance and that cartridge
```

**Invariants** — routing melee through the bullet path is what gives a knife the whole
material system for free: the right impact sound, the right decal and the right armour
interaction on every surface, with no melee-specific code anywhere downstream. The
knife's *material identity* is what the material system keys on, so a blade on concrete
sounds like a blade on concrete.

**Notes** — the magazine is kept topped up to two synthetic rounds so that the weapon
base's "has ammunition" predicate stays true; a knife is never out of ammunition. This is
a hack around the weapon base's assumption that a weapon has a magazine, and a rebuild
should let a weapon declare that it has none.

The gamepad trigger buzz reads the *fire* binding's controller axis twice, so the left
trigger is buzzed on the right trigger's binding. Clearly unintended.

## The targeting search

### `SelectBestHitVictim` — find the sphere and who is in it

**Contract** — establishes the search sphere and the candidate creatures. Requires the
carrier to be the actor with the first-person model active; a stalker's knife always
takes the old single-ray path.

```text
FUNCTION select_best_victim(knife, position) -> bool
  RETURN false unless the carrier is the actor AND the first-person model is active
  camera = the actor's first-person camera frame

  rotate knife.splash_direction into world space by camera
  sphere_centre = position + splash_direction * knife.reach
  sphere        = (sphere_centre, knife.splash_radius)

  candidates = every collideable object whose bounds meet that sphere

  IF this is the THRUST AND candidates is non-empty THEN
    keep only the SINGLE nearest living creature within 2 m of the sphere's centre
  ELSE
    keep every living creature within 2 m of the sphere's centre
  RETURN candidates is non-empty
```

**Invariants** — the splash direction is authored *in camera space*, so the sweep follows
where the player looks rather than where the weapon model points. The primary attack's
default leans slightly downward; the secondary's runs straight ahead.

The two-metre prefetch radius is a second, coarser filter applied on top of the sphere
query, measured to each creature's centre rather than its bounds. It is a fixed constant.

### `SelectHitsToShot` — pick the bone shapes

**Contract** — from the candidate creatures, produce up to `hits_count` world positions
to strike, ordered best first.

```text
FUNCTION select_hits(knife, position) -> list<vector3>
  RETURN empty IF select_best_victim found nothing
  victims = the candidates, as living creatures

  # a direction in camera space that ranks bones by how "on the swing's arc" they are
  basis = (-1, 1, 0) for the slash, (0, -1, 0) for the thrust
  rotate basis into world space by the camera frame; normalize

  shapes = empty
  FOR EACH victim
    FOR EACH collision shape of the victim's skeleton
      project the shape's centre and the sphere's centre onto `basis`
      d = the squared distance between those projections
      IF d < (0.2 for the slash, 0.1 for the thrust) THEN
        record (shape, victim, d)
  sort shapes by d ascending
  RETURN choose_shots(knife, shapes, sphere)
```

**Invariants** — the projection onto a single axis is what makes the slash pick bones
*along the blade's arc* rather than simply the nearest ones: a slash that sweeps
left-to-right prefers a row of bones at similar height, which is what produces the
"cut through two people" feel. The axis and the threshold are the only difference between
the two attacks' target selection, and both are hard-coded.

### `fill_shots_list` — the per-victim cap

**Contract** — walks the sorted shapes, keeps those genuinely intersecting the sphere,
and enforces the per-victim cap.

```text
FUNCTION choose_shots(knife, shapes, sphere) -> list<vector3>
  shots = empty; hits_per_victim = empty
  FOR EACH (shape, victim, _) IN shapes
    RETURN shots IF shots is full                 # capacity is knife.hits_count
    CONTINUE unless shape intersects sphere       # exact test, by shape kind
    IF knife.per_victim_count > 0 THEN
      IF victim has no recorded hits THEN record 1
      ELSE IF victim's recorded hits < per_victim_count THEN increment
      ELSE CONTINUE                               # this victim is full
    append the shape's centre to shots
  RETURN shots
```

**Invariants** — a per-victim count of zero (the thrust) disables the cap entirely, so
every hit may land on the one creature the thrust selected. That is how a thrust
concentrates.

### Shape intersection

**Contract** — three exact sphere-versus-shape tests, one per skeleton shape kind, because
a skeleton's collision proxies are spheres, boxes and cylinders:

- **sphere** — centre distance against the summed radii;
- **box** — inflate the box's half-extents by the query radius and test the query's centre
  against the inflated box in the box's own frame. This is the Minkowski trick, and it
  over-reports near corners (it tests against a rounded box's bounding box, not the
  rounded box) — acceptable for a melee sweep, and cheap;
- **cylinder** — the honest test: reject on axial distance, reject on radial distance,
  accept within the barrel, and otherwise solve the end-cap case by intersecting the
  query sphere with the cap's plane and comparing circle centres.

**Notes** — the cylinder case is the only exact one, and it is exact because a limb is a
cylinder and a miss on a limb is visible. A rebuild may approximate the box and sphere
cases as this one does, but should keep the cylinder exact.

### `TryPick`

**Contract** — a first-hit ray query against objects (not static geometry) along the aim,
up to the attack's reach, ignoring the carrier. The callback stops at the first object
that is not the carrier, so the result is the nearest thing directly in front.

## `Action` and the two attacks

**Contract** — the fire binding starts the primary attack; the **zoom** binding is
rewritten to start the secondary. A knife has no zoom (`IsZoomEnabled` is always false),
so the aim button is free to be the second attack. That is the whole reason the knife
suppresses zoom.

## `LoadFireParams` — the second attack's damage

**Contract** — the weapon base loads one attack's damage; this loads the second's from
parallel keys, in the same per-difficulty list form.

```text
FUNCTION load_fire_params(knife, section)
  base.load_fire_params(section)
  attack 1 = the base's loaded hit power, critical power, impulse, and "hit_type"

  # attack 2's power is a list: master first, then veteran, stalker, novice.
  # A shorter list means the missing difficulties keep the master figure.
  list = section "hit_power_2"
  crit = section "hit_power_critical_2", defaulting to the same list
  attack 2's power at every difficulty = list[0]
  IF list has 2 entries THEN veteran = list[1]
  IF list has 3 entries THEN stalker = list[2]
  IF list has 4 entries THEN novice  = list[3]
  (the same for the critical list)
  attack 2's impulse   = section "hit_impulse_2"
  attack 2's hit type  = section "hit_type_2"
```

**Invariants** — the list is ordered *hardest first*, and a one-entry list means the same
damage at every difficulty. That ordering is frozen by the shipped data.

## `GetBriefInfo`

**Contract** — name and icon only; no ammunition, no fire mode.

## Dead code

`GetVictimPos` is an empty body wrapping a large commented-out implementation that would
have aimed at a named bone rather than at the nearest shapes. The corresponding
`m_SplashHitBone` field is never assigned. A rebuild should drop both.
