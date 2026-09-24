# src/xrGame/Level_bullet_manager_firetrace.cpp

> The half of the bullet manager that decides what a bullet does when its ray meets something: whether the target is worth hitting at all, whether the bullet ricochets, sticks or punches through, how much energy it keeps, and what mark, sound and particle the impact leaves.

**Needs** — [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`Entity.h`](Entity.h.md) · [`Level.h`](Level.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`Actor.h`](Actor.h.md) · [`Weapon.h`](Weapon.h.md) · [`ParticlesObject.h`](ParticlesObject.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`ai/monsters/basemonster/base_monster.h`](ai/monsters/basemonster/base_monster.h.md) · [`character_info.h`](../xrServerEntities/character_info.h.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame ray results for every bullet in flight, with material lookups and randomness in the inner loop; nothing here touches a device

## Purpose

A bullet is a ray marched forward each frame. This file holds the three decisions made
where that ray lands, kept separate from the flight integration in the rest of the bullet
manager because they are the *game* half of ballistics rather than the motion half: a
filter that runs during the ray query and can veto a candidate before it becomes a hit
(`test_callback`); the physics of the impact, which mutates the bullet in place and
returns the damage record (`ObjectHit`); and the audiovisual consequence — wallmark,
sound, particle (`FireShotmark`) — plus the two entry points that fan an already-resolved
impact event into those (`StaticObjectHit`, `DynamicObjectHit`).

The split is real and worth preserving: `ObjectHit` is pure enough to run on a worker,
while the shotmark side allocates renderer and audio resources and must run on the frame
that owns them.

## State

Almost all state lives on the bullet record declared alongside the manager; this file
mutates it. The fields it decides:

```text
RECORD BulletImpactState          # the mutable slice of a bullet this file writes
  bullet_pos      : vec3          # moved to the impact point on a ricochet, nudged forward on a pass-through
  dir             : vec3          # reflected + jittered on ricochet, jittered on pass-through
  speed           : real          # scaled by the outcome; zero means the bullet is spent
  armor_piercing  : real          # only a magnetic beam spends this instead of speed
  allow_ricochet  : bool          # invariant: a bullet ricochets at most once per life
  ricochet_was    : bool          # set on ricochet; disables the shooter-ignore window
  density_mode    : bool          # true while the ray is inside solid material
  density         : real          # the density factor of the material currently being traversed
  begin_density   : vec3          # where the ray entered that material
```

**Invariants**

- `allow_ricochet` is cleared the instant a ricochet happens, so a bullet can deflect once
  and then only stick or punch through. Without this a bullet can bounce between two
  surfaces forever.
- `density_mode` toggles on the *sign of the surface normal against the flight direction*:
  a static triangle whose normal faces away from the bullet is an entry face, one facing
  toward it is an exit face. Entry sets the mode and records the entry point; exit clears
  it. Speed is billed for the distance travelled between them.
- The self-hit window: while the bullet has flown less than the shooter-ignore distance
  **and** has not yet ricocheted, its own shooter is not a valid target. A ricochet
  cancels the exemption, so a bullet bounced off a wall can hit the person who fired it.

## `test_callback`

**Contract** — the per-candidate filter the collision database calls while tracing a
bullet's ray segment. Given the ray, a candidate object and the bullet, it answers
"consider this object for a hit" or "skip it entirely for this query". It has side
effects: on a near-miss against a character it plays the whine sound, and on a
probability-rejected hit it plays it too. It reads the difficulty and weapon tuning, and
consumes randomness, so it is not pure and not safe to run twice for one candidate.

**Invariants** — must not admit the shooter inside the ignore window; must not admit a
dead entity as a probabilistic target (dead bodies fall through to the ordinary
collision-shape path).

```text
FUNCTION test_callback(ray, candidate, bullet) -> bool   # true = test this object
  IF candidate is none                       THEN RETURN true
  IF candidate.id == bullet.parent_id
     AND bullet.fly_dist < SHOOTER_IGNORE_DISTANCE
     AND NOT bullet.flags.ricochet_was       THEN RETURN false

  entity = candidate AS entity OR none
  IF entity is none OR NOT entity.alive OR entity.id == bullet.parent_id
     THEN RETURN true                        # ordinary geometry: let the ray decide

  form = entity.collision_form
  IF form is none OR form.kind != OBJECT_FORM THEN RETURN true

  is_probabilistic = (entity is actor AND game is single-player) OR (entity is stalker)
  IF NOT is_probabilistic                    THEN RETURN true

  # Cheap bounding-sphere reject before the expensive skeleton query.
  sphere = form.bounding_sphere TRANSFORMED BY entity.transform
  IF NOT sphere INTERSECTS ray FROM bullet.bullet_pos ALONG bullet.dir WITHIN ray.range
     THEN RETURN false                       # never test this object again this query

  shooter   = find_object(bullet.parent_id)
  play_whine = true
  admit      = true

  IF entity is actor THEN
    admit, play_whine = actor_hit_probability_roll(bullet, entity, shooter, ray, hit_distance)
  END IF

  IF play_whine THEN
    play_whine_sound(bullet, shooter, at: bullet.bullet_pos + bullet.dir * hit_distance)
  END IF
  RETURN admit
```

### Named step: `actor_hit_probability_roll`

The single load-bearing rule in the filter, and the reason the game feels the way it does
on different difficulties. A shot at the player is not resolved purely geometrically: it
must first *pass a probability roll*, and only then is the player's skeleton actually
ray-tested.

```text
FUNCTION actor_hit_probability_roll(bullet, actor, shooter, ray, out distance)
        -> (admit : bool, play_whine : bool)
  # Base probability comes from the weapon if we can still find it, else from the
  # difficulty setting carried on the actor.
  base = actor.hit_probability
  dist_factor = 1
  weapon = find_object(bullet.weapon_id)
  IF weapon is a weapon THEN
    base        = weapon.hit_probability
    flown       = bullet.fly_dist + distance
    dist_factor = min(1, flown / manager.hit_probability_max_distance)
  END IF

  # At point blank the probability is 1; it falls toward the tuned value with range.
  # This is the inverse of intuition and deliberate: misses are a long-range effect.
  probability = dist_factor * base + (1 - dist_factor) * 1

  # A stalker shooter scales the whole thing by its character's marksmanship.
  shooter_factor = 1
  IF shooter is a stalker THEN shooter_factor = shooter.character.hit_probability_factor

  IF random_unit() > probability * shooter_factor THEN
    RETURN (admit: false, play_whine: true)    # a deliberate miss: whine past the ear
  END IF

  # The roll passed; now geometry has the final say against the actual collision form.
  hit = form.ray_query(ray)
  RETURN (admit: hit, play_whine: NOT hit)
```

**Notes** — returning "skip" rather than "no hit" matters: the collision database drops
the object for the remainder of *this* query, so a rejected character does not block the
ray and the bullet continues to whatever is behind them. A rebuild whose ray query has no
per-candidate veto must instead trace, discard, and re-trace from the rejection point.

The whine sound is the audible trace of a probabilistic miss, which is what keeps the
mechanic from reading as the game ignoring bullets.

## `ObjectHit`

**Contract** — resolves one impact. Given the bullet, the impact point, the ray result and
the target's material index, it computes the surface normal, fills the damage record
(power and impulse) and mutates the bullet: new direction, new position, new speed. It
always returns true — the return exists for callers that expect a veto and no case
produces one. Consumes randomness. Does not allocate, does not touch the renderer.

**Invariants** — energy is never created: the new speed is the old speed times a factor in
`[0,1]`, and the impulse delivered is proportional to the energy actually lost. Power
scales with the bullet's *current* speed over its muzzle speed, so a bullet that has
ricocheted and slowed does less damage.

```text
FUNCTION ObjectHit(bullet, end_point, ray_result, target_material) -> (hit_result, mutated bullet)
  normal = impact_normal(bullet, end_point, ray_result)      # see below

  old_speed    = bullet.speed
  speed_factor = bullet.speed / bullet.max_speed
  hit_result   = bullet.hit_param
  hit_result.power = bullet.hit_param.power * speed_factor

  mtl        = material(target_material)
  mtl_resist = mtl.shoot_factor            # 0 = offers no resistance at all
  ap         = bullet.armor_piercing

  # How much of the bullet's penetration survives this material. Zero means "stopped".
  shoot_factor = 0
  IF ap > 0 AND ap >= mtl_resist THEN shoot_factor = (ap - mtl_resist) / ap

  IF mtl_resist ~= 0 THEN                  # foliage and the like: pass straight through
    RETURN unchanged bullet, hit_result
  END IF

  IF bullet.flags.magnetic_beam AND shoot_factor > 0 THEN
    # A beam loses penetration, not velocity, and is never deflected.
    bullet.armor_piercing = bullet.armor_piercing - mtl_resist * bullet.air_resistance
    RETURN unchanged direction and speed, hit_result
  END IF

  # Ricochet test: reflect, jitter by up to 10 degrees, and compare the jittered
  # reflection against the incoming direction. A grazing hit reflects nearly forward,
  # so the dot product is high; a head-on hit reflects backward and the dot is negative.
  reflected = reflect(bullet.dir, normal)
  candidate = random_direction_within(reflected, 10 degrees)
  grazing   = dot(bullet.dir, candidate)

  IF random_in(0.5, 0.8) < grazing
     AND NOT mtl.flag_no_ricochet
     AND bullet.flags.allow_ricochet THEN
    bullet.flags.allow_ricochet = false          # at most one ricochet per bullet
    scale = 1 - abs(dot(bullet.dir, normal)) * manager.collision_energy_min
    scale = clamp(scale, 0, manager.collision_energy_max)
    speed_scale       = scale
    bullet.dir        = candidate
    bullet.bullet_pos = end_point                # the bullet restarts from the surface
    bullet.flags.ricochet_was = true
  ELSE IF shoot_factor ~= 0 THEN
    speed_scale = 0                              # stuck in the material; the bullet dies
  ELSE
    speed_scale = shoot_factor                   # punched through, carrying that fraction
    bullet.bullet_pos = bullet.bullet_pos + bullet.dir * TINY   # step past the surface
    bullet.dir = random_direction_within(bullet.dir, 2 degrees) # penetration deflects it
  END IF

  bullet.speed = bullet.speed * speed_scale
  energy_lost  = 1 - bullet.speed / old_speed
  hit_result.impulse = bullet.hit_param.impulse * speed_factor * energy_lost
  RETURN true
```

### Named step: `impact_normal`

Two sources, because a dynamic target has no triangle to take a normal from.

```text
FUNCTION impact_normal(bullet, end_point, ray_result) -> vec3
  IF ray_result.object is not none THEN
    # Skeletal target: the "normal" is the outward ray from the struck bone's centre to
    # the impact point. It only has to point the impact particles somewhere plausible.
    IF object has a skeleton collision form THEN
      n = end_point - centre_of_element(ray_result.element)
      RETURN normalize(n), or -bullet.dir when that vector is degenerate
    END IF
  ELSE
    # Static geometry: the true face normal of the struck triangle, and the point where
    # the traversed-material bookkeeping happens.
    n = face_normal(static_triangle(ray_result.element))
    IF bullet.density_mode THEN
      # Bill the bullet for the distance it just spent inside solid material.
      travelled = distance(bullet.begin_density, bullet.bullet_pos + bullet.dir*ray_result.range)
      bullet.speed = max(0, bullet.speed - travelled * bullet.density)
    END IF
    IF dot(n, bullet.dir) < 0 THEN           # entering a solid
      bullet.density_mode = true
      bullet.density      = material(target_material).density_factor
      bullet.begin_density = bullet.bullet_pos + bullet.dir * ray_result.range
    ELSE                                     # leaving it
      bullet.density_mode = false
    END IF
    RETURN n
  END IF
```

**Notes** — the ricochet is faked: the bullet is left sitting exactly on the surface it
bounced off and the ray query is allowed to terminate there, rather than continuing the
remaining segment length in the new direction. A frame of travel is lost. This is
acceptable at the frame rate the engine targets and a rebuild may do it properly, but the
visible consequence is that a ricochet loses a little range.

The two collision-energy constants are manager-level tuning read from configuration: the
minimum scales how much a head-on graze costs, the maximum caps the speed a ricochet can
retain. A perpendicular-to-normal graze keeps almost all its speed; a near-head-on
ricochet keeps almost none.

The 0.5-to-0.8 random ricochet threshold, rather than a fixed angle, means a grazing angle
ricochets *usually* and a moderate angle ricochets *sometimes* — the randomness is the
whole point, and a fixed threshold makes bullet behaviour look mechanical.

## `FireShotmark`

**Contract** — the audiovisual consequence of an impact. Looks up the material *pair*
(bullet material against target material) and, from it, places a decal, plays a collision
sound at the impact point and spawns an impact particle effect. Takes a flag that
suppresses the mark and sound while still allowing the explosive-bullet effect. Allocates
(a particle object, handed to the persistent game layer's play queue), touches the
renderer and the audio device, and must therefore run on the frame thread.

```text
FUNCTION FireShotmark(bullet, dir, end_point, ray_result, target_material, normal, show_mark = true)
  pair = material_pair(bullet.bullet_material_idx, target_material)

  IF ray_result.object is not none THEN          # dynamic target
    particle_dir = -dir
    IF ray_result.object is the camera's own entity THEN RETURN   # no decals on yourself
    IF ray_result.object renders as a first-person model THEN RETURN
    IF pair has collide marks AND show_mark AND skeleton-wallmarks enabled
       AND this process renders at all THEN
      # Placed slightly short of the surface so it does not z-fight the skin.
      p = bullet.bullet_pos + bullet.dir * (ray_result.range - SMALL)
      add_skeleton_wallmark(object transform, object skeleton, pair.marks, p,
                            bullet.dir, bullet.wallmark_size)
    END IF
  ELSE                                            # static geometry
    particle_dir = normal
    IF pair has collide marks AND show_mark THEN
      add_static_wallmark(pair.marks, end_point, bullet.wallmark_size, struck triangle)
    END IF
  END IF

  IF pair has collide sounds AND show_mark THEN
    pick one at random, attach it to the bullet, play it at end_point
    # owned by the shooter so that distance attenuation uses the right listener context
  END IF

  target_is_static = NOT material(target_material).flag_dynamic
  ps_name = one of pair.collide_particles at random, or none

  IF (ps_name AND show_mark) OR (bullet.flags.explosive AND target_is_static) THEN
    # Build an orientation whose forward axis is the impact direction; the other two
    # axes are arbitrary but must be orthonormal, since the particle system rotates by it.
    frame = orthonormal_basis_with_forward(particle_dir) at end_point
    IF ps_name AND show_mark THEN
      spawn a self-removing particle object at frame and queue it on the persistent layer
    END IF
    IF bullet.flags.explosive AND target_is_static THEN play the explode effect at frame
  END IF
```

**Notes** — the collision sound is stored *on the bullet record* before being played. That
is not an optimization; it is lifetime management. The sound reference must outlive the
call, and the bullet is the nearest thing with the right lifetime.

Skeleton wallmarks are gated by a renderer console variable because projecting a decal
onto a skinned mesh costs far more than onto a static triangle, and on the oldest render
path it is off by default.

## `StaticObjectHit`

**Contract** — the impact event handler for a hit on level geometry. A pure delegation to
`FireShotmark` with the event's fields; the damage side has already been resolved and
there is nobody to send a hit message to.

## `DynamicObjectHit`

**Contract** — the impact event handler for a hit on an object. Decides whether to show a
mark at all, computes the impact position in the struck bone's local frame, and emits the
damage message that actually applies the hit. Emits a network event; in single player the
same path runs locally. Does not allocate beyond the message buffer.

**Invariants** — a hit is sent at most once per impact event; the bone-local position must
be expressed in the frame of the *same* bone index that the message carries, or the damage
zone lookup on the receiving side lands on the wrong body part.

```text
FUNCTION DynamicObjectHit(event)
  IF event.target is an entity AND NOT entity.in_solid_state THEN RETURN
     # A ragdolling or otherwise non-solid entity absorbs nothing.

  IF single player THEN event.Repeated = false    # repeat suppression is a multiplayer concern

  show_mark = true
  IF target is an actor AND that player is flagged invincible THEN show_mark = false
  ELSE IF target is a monster THEN show_mark = monster.wants_shotmarks

  FireShotmark(event.bullet, event.bullet.dir, event.point, event.ray_result,
               event.tgt_material, event.normal, show_mark)

  # Express the impact where the damage model can use it: first into the object's local
  # frame, then into the struck bone's frame. A bone-local point survives the object
  # moving between the hit being computed and being applied.
  p_object = inverse(target.transform) APPLIED TO event.point
  IF target has a skeleton THEN
    p_bone = inverse(bone_transform(event.ray_result.element)) APPLIED TO p_object
  ELSE
    p_bone = p_object
  END IF

  IF event.bullet.flags.allow_sendhit AND NOT event.Repeated THEN
    collect_stats = (multiplayer AND target is an actor AND weapon statistics are being collected)
    IF collect_stats THEN record this bullet against the target and bone

    hit = HitRecord(power:    event.hit_result.power,
                    dir:      event.bullet.dir,
                    bone:     event.ray_result.element,
                    position: p_bone,
                    impulse:  event.hit_result.impulse,
                    type:     event.bullet.hit_type,
                    armor_piercing: event.bullet.armor_piercing,
                    aimed:    event.bullet.flags.aim_bullet)
    hit.event_kind = (collect_stats ? HIT_WITH_STATISTICS : HIT)
    hit.target = target.id
    hit.who    = event.bullet.parent_id
    hit.weapon = event.bullet.weapon_id
    hit.bullet = event.bullet.id
    send_as_entity_event(hit)
  END IF
```

**Notes** — damage is *not* applied by calling the target. It is packed into an entity
event message and sent, even in single player, so that one code path serves both. The
consequence a rebuild must accept is that damage lands a message-dispatch later than the
impact, which is why the bone-local position is carried rather than the world point.

The "repeated" flag exists because a multiplayer client can resolve the same bullet
against the same object across two frames; only one hit may be billed. Single player has
one authority and clears it unconditionally.

The statistics variant of the hit event is a *different event identifier*, not a flag on
the same one, because the server-side statistics collector subscribes to it separately.
