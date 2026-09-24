# src/xrGame/entity_alive.cpp

> The base of everything that can be hurt: condition and wounds, blood and burning, faction relations, death, and the surface a bullet is allowed to land on.

**Needs** — [`entity_alive.h`](entity_alive.h.md) · [`entity_alive_inline.h`](entity_alive_inline.h.md) · [`Entity.h`](Entity.h.md) · [`EntityCondition.h`](EntityCondition.h.md) · [`Wound.h`](Wound.h.md) · [`Hit.h`](Hit.h.md) · [`damage_manager.h`](damage_manager.h.md) · [`ParticlesPlayer.h`](ParticlesPlayer.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`monster_community.h`](monster_community.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`material_manager.h`](material_manager.h.md) · [`PHMovementControl.h`](PHMovementControl.h.md) · [`PHDestroyable.h`](PHDestroyable.h.md) · [`Level.h`](Level.h.md) · [`Inventory.h`](Inventory.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`xrServerEntities/game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`xrServerEntities/xrMessages.h`](../xrServerEntities/xrMessages.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`xrEngine/xr_collide_form.h`](../xrEngine/xr_collide_form.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrScriptEngine/script_callback_ex.h`](../xrScriptEngine/script_callback_ex.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)

**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: the substance is scored selection, timers and geometry over a skeleton; only the wallmark ray pick and the physics-support delegation touch a seam

## Purpose

Every creature in the game — the player, stalkers, mutants, anything with health — derives
from this class, and it is the single place that answers four questions the rest of the game
asks constantly:

1. **What happens when this is hit?** The bone's damage multipliers are applied, the
   condition model turns the hit into health loss and possibly a wound, the wound decides
   whether the creature starts bleeding or burning, blood is sprayed onto the world, and the
   attacker's reputation is adjusted.
2. **Who is this creature's enemy?** Not per creature: per *species*. Relations are a table
   between biological communities, and every creature's relations are that table's answer.
3. **What does it mean for this to die?** A specific, ordered shutdown that must also happen
   correctly for a creature killed by bleeding or radiation with no killer present.
4. **Where on this body can something land?** A bone chosen in proportion to its collision
   surface area, and a point chosen uniformly on that bone's shape — which is what makes AI
   aiming at a body distribute over the body rather than at its origin.

The class sits between the generic entity (identity, network, save) and the specialised
creatures. It is a real seam in the hierarchy, not an arbitrary split: everything here
depends on having a *condition model* and a *skeleton with collision shapes*, and nothing
above it does.

## State

```text
RECORD EntityAlive EXTENDS Entity
  entity_condition    : EntityCondition      # health, bleeding, radiation, wounds — owned
  material_manager    : MaterialManager      # footstep and contact sounds — owned
  monster_community   : MonsterCommunity     # which species; the key into the relation table
  mobility            : bool                 # can this creature move at all
  accuracy            : real                 # reset to 25 on reinit
  intelligence        : real                 # reset to 25 on reinit
  squad_index         : int (8-bit)          # 255 = not in a squad
  is_agresive         : bool                 # behaviour flags read by the planner
  is_start_attack     : bool
  eating              : bool                 # a creature is currently feeding on this corpse
  used_time           : int (ms)             # when feeding last stopped
  use_timeout         : int (ms), default 5000   # how long a corpse stays locked after that
  particle_wounds     : list<Wound>          # wounds currently emitting fire effects (borrowed)
  blood_wounds        : list<Wound>          # wounds currently dripping (borrowed)
  ef_creature_type    : int                  # world-state evaluator classification
  ef_weapon_type      : int                  # 'unset' sentinel = all bits set; reading it then fails
  ef_detector_type    : int                  # same
  hit_bone_areas      : list<(bone, area)>   # cached, sorted by area descending
  hit_bone_areas_valid: bool                 # invalidated whenever the visual changes
  hit_bones_random    : random stream        # private stream, see Notes
```

Shared across **all** living entities, loaded once from configuration by the first one
constructed and released at shutdown:

```text
RECORD BloodPresentation (static)
  wallmark_textures   : wallmark array       # splashes on walls
  drop_textures       : wallmark array       # drips under a bleeding creature
  mark_size_min/max   : real
  mark_distance       : real                 # how far blood flies before it stops looking for a wall
  nominal_hit         : real                 # the damage that produces a full-size mark
  start/stop_blood_wound_size : real         # hysteresis on "is this wound dripping"
  drop_size           : real

RECORD FirePresentation (static)
  particle_names      : list<text>           # one is chosen at random per burning wound
  start/stop_burn_wound_size : real          # hysteresis on "is this wound burning"
  min_burn_time       : int (ms)
```

Invariants:

- **The wound lists borrow.** The condition model owns every wound; these two lists are
  subsets of it, and an entry is removed when the wound reports itself destroyed. Nothing
  here frees a wound.
- **Start and stop thresholds are deliberately different** (0.3 and 0.1 by default in both
  the blood and the fire case). A single threshold would make a wound hovering at the
  boundary start and stop its effect every update. The gap is hysteresis, and it is the
  reason two numbers exist where one would do.
- **`hit_bone_areas` is a cache keyed on the visual.** Changing the model invalidates it; it
  is rebuilt on the next use, not eagerly.
- **The weapon and detector classifications have an 'unset' sentinel and reading an unset
  one is a hard failure.** Not every creature has them; the ones that do must have them
  configured, and a silent zero would mis-classify the creature to the world-state
  evaluator.

**Notes** — the static presentation tables are loaded lazily by whichever creature is
constructed first and are never reloaded. That is a global initialised from data, with the
usual consequence: the values are frozen for the process and a configuration reload does not
reach them. A rebuild should load them with the rest of the configuration.

## `Hit`

**Contract** — the entry point for all damage. Takes a hit descriptor (damage, direction,
struck bone, position in bone space, hit type, attacker). Normalises the hit type, applies
the bone's multipliers, hands it to the condition model, starts the resulting presentation,
sprays blood, and registers the attack with the reputation system. Mutates a **copy** of the
descriptor, so the caller's is untouched except where the base class writes back.

```text
FUNCTION hit(descriptor)
  h = copy of descriptor
  IF h.type is the second wound type: h.type = wound          # see Notes

  # the per-bone multipliers land in the condition model, not in the descriptor
  (h_scale, w_scale) = damage_manager.hit_scale(h.bone, h.aim_bullet)
  conditions.hit_bone_scale   = h_scale
  conditions.wound_bone_scale = w_scale

  wound = conditions.apply(h)          # health, bleeding, radiation; may create a wound
  IF wound exists
    IF h.type is burn or light burn: start_fire_particles(wound)
    ELSE IF h.type is wound or fire wound: start_blood_drops(wound)

  IF h.type is not telepathic
    bloody_wallmarks(h.damage, h.direction, h.bone, h.position_in_bone_space)

  conditions.condition_delta_time = 0   # see Notes
  base.hit(h)

  IF alive AND single-player
    attacker = h.who as a living entity
    IF attacker exists AND attacker is alive AND attacker is not me
      relations.register_fight(attacker, me, relation_to(attacker), h.damage)
      relations.record_action(attacker, me, ATTACK)
```

**Invariants** — the condition model is updated **before** the base class sees the hit. The
base class's own handling (death detection, network propagation) reads the post-hit health,
so reversing the order would make an entity die a frame late and, worse, report the wrong
health over the network.

**Notes** — three details are load-bearing and none is obvious.

*The second wound type is folded into the first.* There are two wound hit types in the
enumeration and they are identical from here on; the distinction exists earlier, in how the
hit was produced. Folding it here means every downstream test compares against one value. A
rebuild with a cleaner hit-type set should collapse them at the source instead.

*The bone multipliers are pushed into the condition model rather than applied to the
damage.* The condition model needs them separately because a hit scales health loss and
bleeding by different factors, and it decides per damage channel which applies. Handing it
two numbers is how that is expressed.

*The condition's elapsed-time accumulator is zeroed.* The condition model advances health,
bleeding and radiation by elapsed time; zeroing the accumulator at the moment of a hit stops
a creature that has not been updated for a while from immediately applying a large
time-based change on top of the hit. A creature hit while the scheduler had it parked would
otherwise take a visible extra chunk of bleeding damage the instant it is struck.

Telepathic damage leaves no blood — it does not break skin. That is the only hit type
excluded from the wallmark path.

Reputation is registered only in single player and only between two living, distinct
entities. Both clauses matter: multiplayer has no reputation model, and a creature damaging
itself (falling, its own grenade) must not become its own enemy.

## `Die`

**Contract** — the death sequence, in a fixed order. Records the kill with the reputation
system, runs the base class's death handling, fires the script death callback, tells the
authoritative side who the killer was, stops the corpse reacting to sound, and hands the
body to the physics support. Called once; the caller guarantees that.

```text
FUNCTION die(killer)
  IF single-player: relations.record_action(killer, me, KILL)
  base.die(killer)
  script_callback(DEATH)(me, killer)                 # killer may be absent

  IF NOT being destroyed AND single-player
    event = new event(ASSIGN_KILLER, destination = my id)
    event.write_entity_id(killer.id)
    send(event)

  spatial_flags = spatial_flags without REACT_TO_SOUND   # see Notes
  IF character physics support exists: physics_support.on_die()
```

**Invariants** — the order is load-bearing at two points. The reputation kill is recorded
*before* the base class's handling, because that handling can begin destroying the entity
and the relation lookup needs it intact. And the script callback fires *after* the base
class's handling, so a script reading the creature's state in its death callback sees a dead
creature, not a dying one.

**Notes** — clearing the react-to-sound flag is the one line here that changes gameplay: a
corpse must stop being a sound receiver, or the perception system keeps delivering events to
something that will never act on them, and the cost is paid every time a shot is fired near
a battlefield full of bodies. It is done by clearing a bit in the spatial index's type mask
rather than by unregistering, because the body is still a renderable and a collision target.

The killer event is sent only in the single-player game, where the same process is both
sides, and only when the entity is not already being destroyed. It exists so the
authoritative record of the entity knows who killed it — which survives into the save and
into the alife simulation's account of the world.

A creature that dies of bleeding or radiation with no attacker reaches this path from the
scheduled update (below) with itself as the killer.

## `shedule_Update`

**Contract** — the per-update pass for a living entity: advance the condition model by
elapsed game time, update the two presentation lists, age the wounds, and kill the entity if
its condition says it should be dead. Runs on the owning side only for the kill.

```text
FUNCTION scheduled_update(delta)
  base.scheduled_update(delta)
  conditions.update_condition_time()     # advance the clock the condition integrates over
  conditions.update_condition()          # health, bleeding, radiation, satiety…
  update_fire_particles()
  update_blood_drops()
  conditions.update_wounds()             # heal or deepen each wound

  IF locally simulated AND NOT alive AND NOT already dying
    IF conditions knows who hit last: kill_entity(that entity)
    ELSE                              kill_entity(me)
```

**Invariants** — the condition's *time* advance precedes its *state* advance, which precedes
the presentation updates, which precede the wound ageing. Reversing any pair produces a
one-update lag in something visible: effects on a wound that no longer exists, or a wound
healed before its damage was counted.

**Notes** — death is detected here, not in the hit path, and that is the whole reason this
ordering exists. A creature can die from bleeding, radiation, starvation or a hit; only the
last of those has an obvious killer. Routing every death through one place, with "who hit me
last" as the killer and *myself* as the fallback, means the death sequence is identical for
all of them and the reputation, save and script paths never have to special-case a
killerless death.

The three-way guard — locally simulated, not alive, not already dying — is the standard
"exactly once, on the owning side" shape. In multiplayer only the authoritative side runs
it; in single player both sides are the same process.

## `net_Spawn`

**Contract** — reinitialises the condition model, runs the base spawn, clears both
presentation lists and then **re-derives them from the wounds the condition model was
restored with**. Yields success unconditionally.

```text
FUNCTION net_spawn(spawn_record)
  conditions.reinit()
  base.net_spawn(spawn_record)
  blood_wounds.clear() ; particle_wounds.clear()
  FOR EACH wound IN conditions.wounds
    start_fire_particles(wound)      # each re-tests its own threshold
    start_blood_drops(wound)
  RETURN success
```

**Invariants** — the condition model is reinitialised *before* the base spawn because the
base spawn path reads health to decide whether the entity spawns alive. The presentation
rebuild happens *after*, because the wounds only exist once the spawn record has been
applied.

**Notes** — this is the load-restoration path as much as the spawn path. A creature loaded
from a save with three open wounds must come back bleeding, and it does because each wound
is re-offered to the same two threshold tests a fresh hit would go through. Nothing about
the presentation is serialised; it is entirely derived from wound state, which is the right
call — the effects are ephemeral and the wounds are not.

## `BloodyWallmarks` / `PlaceBloodWallmark`

**Contract** — sprays blood from a hit onto whatever is behind the creature. Takes the
damage, the hit direction, the struck bone and the hit position in bone space. Does nothing
for an unattributed hit. Traces from the world-space hit position along the hit direction;
marks only static geometry whose material is flagged as accepting blood.

```text
FUNCTION bloody_wallmarks(damage, direction, bone, position_in_bone_space)
  IF bone is none: RETURN
  start = position_in_bone_space
        through the bone's current pose
        through the entity's world transform

  size = mark_size_max * (damage / nominal_hit)
  IF entity radius < 0.6 units: size = size * 0.5        # see Notes
  clamp size to [mark_size_min, mark_size_max]
  place_blood_wallmark(direction, start, mark_distance, size, wallmark_textures)

FUNCTION place_blood_wallmark(direction, start, distance, size, textures)
  hit = collision.ray_pick(start, direction, distance, static and dynamic, ignoring self)
  IF no hit OR the hit was a dynamic object: RETURN       # see Notes
  triangle = the static triangle that was hit
  IF NOT triangle.material.accepts_bloodmarks: RETURN
  point = start + direction * hit.range
  renderer.add_static_wallmark(textures, point, size, triangle)
```

**Notes** — the mark size is proportional to damage against a *nominal* hit, clamped to a
configured range, and halved for small creatures. The nominal hit is what makes the scale
meaningful: it is the damage value that produces a full-size mark, so the artist tunes one
number and every weapon's marks scale from it. The small-entity threshold of 0.6 units of
radius separates rats and dogs from people; it is a magic constant with no derivation, and
its effect is that a small animal does not paint a human-sized splash.

The ray is cast with both static and dynamic geometry enabled but a dynamic hit is then
*rejected*. That is not wasted work: including dynamic geometry in the pick means blood
stops at a body or a crate standing between the creature and the wall instead of painting
the wall behind it. Blood simply does not stick to dynamic objects, so the mark is dropped.

The material flag is the authoring control. Blood marks water, concrete and wood; it does
not mark surfaces the material library says it should not.

The same placement routine serves both splashes and drips — the callers differ only in
direction, size and texture set.

## `StartFireParticles` / `UpdateFireParticles`

**Contract** — a wound whose burn component exceeds the start threshold begins emitting a
fire effect at the nearest particle attachment bone above the wounded bone. The update pass
retires effects for wounds that have been destroyed, have burnt down below the stop
threshold, or whose owner has died.

```text
FUNCTION start_fire_particles(wound)
  IF wound.size(burn) <= start_burn_size: RETURN
  IF wound not already in particle_wounds: particle_wounds.append(wound)
  wound.particle_bone = nearest_particle_bone_above(wound.bone)
  wound.particle_name = one of particle_names, chosen at random
  duration = min_burn_time * random(0.5 .. 1.5)
  IF wound.particle_bone exists
    start_particles(wound.particle_name, wound.particle_bone, up, my id, duration, no_auto_stop)
  ELSE
    start_particles(wound.particle_name, up, my id, duration, no_auto_stop)   # every bone

FUNCTION update_fire_particles()
  IF particle_wounds is empty: RETURN
  FOR EACH wound IN particle_wounds
    size = wound.size(burn)
    IF wound.destroyed OR (size > 0 AND (size < stop_burn_size OR NOT alive))
      auto_stop_particles(wound.particle_name, wound.particle_bone,
                          min_burn_time * random(0.5 .. 1.5))
      remove wound from particle_wounds
```

**Notes** — the effect is started with auto-stop *disabled* and a duration anyway, then
given a *second* randomised duration when it is retired. So the fire keeps burning for a
randomised tail after the wound stops qualifying, rather than snapping out. Both durations
are `min_burn_time` scaled by an independent random factor between a half and one and a
half, which means the effective minimum burn time is half the configured one — the name is
misleading and a rebuild should not read it as a floor.

The effect name is drawn uniformly from the configured list per wound, so two burning wounds
on one creature look different. The random index is drawn over the full size of the list,
and the source's range call is inclusive at the top on some readings — a detail worth
checking in a rebuild, since an off-by-one here indexes past the list.

The per-bone path falls back to *every* attachment bone when the wound's bone has no
particle attachment above it. That is the loud fallback: a creature burning everywhere is
obviously wrong and gets noticed, where silence would not.

A dead creature stops burning. That is a deliberate suppression, not a consequence — the
wounds are still there, but a corpse does not emit fire effects.

## `StartBloodDrops` / `UpdateBloodDrops`

**Contract** — a wound whose bleeding exceeds the start threshold is added to the drip list
with its drip timer armed. The update pass drips blood onto the ground beneath the wound at
an interval that shortens as the wound worsens, and retires wounds that close or whose owner
dies.

```text
FUNCTION start_blood_drops(wound)
  IF wound.blood_size <= start_blood_size: RETURN
  IF wound not already in blood_wounds
    blood_wounds.append(wound)
    wound.next_drop_time = 0            # drip immediately

FUNCTION update_blood_drops()
  IF blood_wounds is empty: RETURN
  IF NOT alive: blood_wounds.clear() ; RETURN        # corpses do not drip

  FOR EACH wound IN blood_wounds
    IF wound.destroyed OR wound.blood_size < stop_blood_size
      remove wound ; CONTINUE
    IF wound.next_drop_time >= global_clock: CONTINUE

    severity = min(wound.blood_size - stop_blood_size, 1)
    interval = drop_time_max - (drop_time_max - drop_time_min) * severity
    wound.next_drop_time = global_clock + interval * random(0.8 .. 1.2)

    IF wound.bone exists
      position = world position of wound.bone
               + a random direction scaled by 0.15 units
      place_blood_wallmark(straight down, position, mark_distance, drop_size, drop_textures)
```

**Notes** — the drip interval is a linear interpolation between a configured maximum and
minimum, driven by how far the wound's bleeding exceeds the stop threshold, capped at one.
So a barely-open wound drips at the slow rate and anything past one unit of excess bleeding
drips at the fast rate; the whole dynamic range of the effect is that one unit. The extra
random factor of plus or minus twenty percent stops several wounds on one creature dripping
in lockstep.

The 0.15-unit random offset around the bone scatters the drips so a standing bleeding
creature leaves a patch rather than a stack of identical decals in one spot.

Drips go straight down and use the same trace distance as splashes, so a creature bleeding
on a catwalk marks the floor below it.

The two interval bounds are read from configuration **once, into function-local statics**,
so they are captured on the first call and frozen. Incidental, and it means a configuration
reload does not reach them — the same wart as the static presentation tables.

## `tfGetRelationType` / `is_relation_enemy`

**Contract** — the faction relation between this creature and another, and the boolean
predicate every AI enemy test funnels through. Relations are looked up in the community
relation table by the two creatures' species indices, not by their identities. Pure.

```text
FUNCTION relation_to(other) -> Relation
  SWITCH community_relation(my species, other species)
    1  -> friend
    0  -> neutral
   -1  -> enemy
   -2  -> worst enemy
    otherwise -> dummy

FUNCTION is_relation_enemy(other) -> bool
  RETURN relation_to(other) is enemy OR worst enemy
```

**Notes** — the numeric encoding is a small signed scale where negative is hostile and more
negative is more hostile, translated here into named values. Keeping the raw scale in the
table and the names in the code is a reasonable split; a rebuild may store the names
directly, but the *shipped data* holds the numbers, so the mapping is frozen.

"Worst enemy" is distinguished from "enemy" in the enumeration and, at this level, treated
identically. The distinction matters higher up, where it affects target priority.

The dummy value for an out-of-range relation is a soft failure: an unknown relation is
neither friend nor enemy, so a data error produces neutrality rather than a crash or a
massacre.

Personal relations — one specific stalker's opinion of another — are **not** here. This is
the species-level default; the reputation registry layers individual history over it, and
derived classes override this method to consult it.

## `get_new_local_point_on_mesh`

**Contract** — chooses a random point on the creature's *collision surface*, weighted by
area, and reports both the point (in model space) and the bone it belongs to. This is what
an AI shooting at a body aims at. Falls back to the base class's simpler answer whenever
there is no skeleton, no bones, no pickable shapes, or the old-vision flag is set. Uses a
private random stream. Rebuilds the area cache on first use after a visual change.

```text
FUNCTION random_point_on_mesh() -> (point, bone)
  IF old_vision_flag OR no skeleton OR no bones: RETURN base.random_point_on_mesh()
  IF cache not valid: fill_hit_bone_surface_areas()
  IF cache empty: RETURN base.random_point_on_mesh()

  # Pass 1 — total area of the VISIBLE pickable bones
  total = sum of area over cached bones whose bone is currently visible
  target = random in [0, total)

  # Pass 2 — walk the same order accumulating until the target is passed
  FOR EACH (bone, area) IN cache
    IF bone not visible: CONTINUE
    accumulator = accumulator + area
    IF accumulator >= target: BREAK

  RETURN (random_point_on_shape(bone.shape), bone)
```

**Invariants** — both passes must skip invisible bones *identically*, or the accumulator
can never reach the target and the walk runs off the end. Bone visibility changes at
runtime (a limb blown off, a model with hidden parts), which is why the total cannot be
cached alongside the areas.

**Notes** — weighting by surface area is the decision that makes AI fire look right. A
uniform choice over bones would send as many shots at a finger as at a torso; weighting by
area sends most shots at the big shapes and occasionally at a limb, which is what a human
aiming centre-of-mass with imperfect aim produces. The shipped alternative — the old-vision
flag, which falls back to the base class's single point — is kept as an escape hatch and
defaults off.

The private random stream is used for the bone choice while the *point within* the shape
uses the global one. Splitting the bone choice onto its own stream makes the distribution of
targeted bones independent of how much other randomness the frame consumed, which matters
because this is called from perception rather than from simulation and must not be
perturbed by unrelated work. The inconsistency — one stream for the bone, another for the
point — looks unintentional.

The cache is sorted by area descending. Nothing in the walk requires sorted order, so the
sort buys only that the common case exits the second pass after a few iterations.

## `fill_hit_bone_surface_areas`

**Contract** — private. Recomputes the per-bone surface-area table from the skeleton's
collision shapes, skipping bones with no shape and bones flagged unpickable, then sorts
descending by area. Requires a skeleton with at least one bone.

```text
FUNCTION fill_hit_bone_surface_areas()
  cache.clear()
  FOR EACH bone
    shape = bone.collision_shape
    IF shape is none OR shape is flagged not-pickable: CONTINUE
    SWITCH shape.kind
      box      -> area = 2 * (hx*(hy+hz) + hy*hz)      # from half-extents
      sphere   -> area = 4 * pi * r^2
      cylinder -> area = 2 * pi * r * (r + h)          # see Notes
    cache.append(bone, area)
  sort cache by area descending
  cache_valid = true
```

**Notes** — the cylinder formula is the full closed cylinder: lateral surface plus both end
caps. Keeping the caps in the area is consistent with the point-selection code below, which
can place a point on an end cap.

The unpickable flag is the authoring control for "this bone is part of the skeleton but is
not a target" — attachment points, helper bones. Respecting it here is what stops AI aiming
at a weapon-mount bone.

Invalidated by a visual change rather than recomputed, so a creature that swaps models pays
the rebuild on its next perception query, not at the swap.

## Random point on a collision shape

**Contract** — part of the point selection above. Produces a point on the *surface* of the
chosen shape, in the shape's own frame, then transformed into model space.

- **Box** — one of the six faces is chosen uniformly, and a point on it uniformly; the code
  achieves this by fixing one coordinate to a face and randomising the other two, then
  rotating the coordinate triple by the chosen face's axis.
- **Sphere** — a uniform random direction scaled by the radius, offset by the centre.
- **Cylinder** — one draw decides between the lateral surface and an end cap, proportioned
  by height against radius; the angle around the axis is a second uniform draw.

**Notes** — the box case picks a *face uniformly*, not proportionally to that face's area.
For a long thin box — a limb — that over-samples the two small end caps. It is a real
distribution error, small in effect, and worth fixing rather than reproducing.

The cylinder case builds its frame from a normal constructed as the component-wise
difference of the axis direction's coordinates, which is a cheap way to get *some* vector
not parallel to the axis. It degenerates when the axis has all three components equal; no
shipped shape does, and a rebuild should use an explicit perpendicular construction rather
than copy the trick.

The cap-versus-side split compares a draw over (height + radius) against the height, so the
proportion is height-to-radius rather than the true lateral-to-cap area ratio, which would
be height-to-(radius/2)... The same kind of approximation as the box. Both are *plausible*
distributions rather than correct ones, and the game does not notice.

## `get_last_local_point_on_mesh`

**Contract** — takes a point previously chosen on a bone and re-derives its current world
position from that bone's current animation pose. Falls back to the base class when no bone
was recorded.

**Notes** — this is the companion to the point selection above and it is what makes aiming
at a moving body track. The point is chosen once, in bone space, and then follows the
animation: a creature aiming at a running target's chest keeps aiming at the chest rather
than at a stale world point. That is the entire reason the point is stored in bone space
and the bone identifier is carried alongside it.

## `Load` / `reload` / `reinit`

**Contract** — `Load` reads the creature's condition parameters, its immunity table (named
by a second section), its mass-derived food value and its species, and triggers the one-time
load of the shared blood and fire presentation tables. `reload` re-reads only the
world-state classifications and the food value. `reinit` resets accuracy and intelligence to
25.

**Notes** — food value is a hundred times the physical mass. That is the number the alife
simulation uses to decide how long a corpse feeds a mutant, and tying it to mass rather than
authoring it separately means a bigger creature is automatically a bigger meal.

The immunity table lives in a *separate* section named from this one, so several creatures
share one immunity profile. The species likewise is a name into the community table.

`reinit`'s 25 for accuracy and intelligence is a magic pair with no discoverable
justification — the fields are read by derived behaviour and the value is simply a
mid-range default.

## `save` / `load` / `net_SaveRelevant`

**Contract** — serialisation appends the condition model's state after the base entity's,
and reads it back in the same order. Every living entity is save-relevant unconditionally.

**Notes** — this is the frozen half of the class. The condition model's field order is part
of the save format and part of the network spawn payload; see the persistence section of the
system requirements. Nothing about wounds' *presentation* is written, only the wounds
themselves.

## `create_entity_condition`

**Contract** — creates the condition model, or adopts one a derived class has already made,
and hands it to the base class. Called during construction.

**Notes** — the "adopt if given" shape is how a derived creature substitutes a richer
condition model (the player's, with stamina and satiety) without this class knowing about
it. A rebuild expresses the same thing as a factory method on the derived class.

## Physics-support delegation

**Contract** — a band of methods that forward to the character physics support when it
exists: the synchronisation item count and accessors, freeze and unfreeze, linear velocity,
the inverse-kinematics limb controller, the contact sound player, and the collision hit
callback pair. Each falls back either to the base class or to nothing when no physics
support exists.

```text
FUNCTION freeze()
  IF physics_support.movement has a character body: physics_support.movement.freeze()
  ELSE IF a ragdoll shell exists:                   shell.freeze()
```

**Notes** — the two-way fallback in freeze and unfreeze is the live/dead split: a living
creature is a character controller and a dead one is a ragdoll, and they are frozen through
different handles. A rebuild must keep both paths, because entities are frozen and thawed
across level transitions in both states.

One forwarder — the contact sound player — recurses infinitely when there is no physics
support. It is a genuine bug, reachable only for a living entity with no character physics,
which the shipped data never produces.

The hit impulse method is entirely commented out, so a hit applies no physical push at this
level; whatever push a hit produces comes from elsewhere. The disabled body divided the
impulse by the creature's mass, which tells you the *intent*: hit impulse is a force, not a
velocity, and a rebuild reinstating it should scale by mass.

## `set_lock_corpse` / `is_locked_corpse`

**Contract** — the corpse-contention lock. A creature that begins feeding on a body marks it
locked; when it stops, the body stays locked for a timeout (five seconds by default) before
becoming available again.

```text
FUNCTION set_lock_corpse(locked)
  IF currently eating AND NOT locked: used_time = global_clock   # start the cool-down
  eating = locked

FUNCTION is_locked_corpse() -> bool
  IF NOT eating AND used_time + use_timeout > global_clock: RETURN true
  RETURN eating
```

**Notes** — the cool-down is what stops two mutants fighting over a body: the second one
sees it locked for five seconds after the first walks away and goes elsewhere rather than
immediately taking its place. The timestamp is stamped on *release*, not on acquisition,
which is the whole trick.

The initial timestamp is the global clock at construction, so a freshly spawned corpse is
locked for the first five seconds of its existence. Harmless, and probably unintended.

## `net_Relcase`

**Contract** — clears every reference to a destroyed object, including the condition model's
"who hit me last". Must be called for every destruction.

## `CalcCondition` / `g_Radiation` / `SetfRadiation`

**Contract** — `CalcCondition` updates the condition model and reports health lost,
deliberately **not** calling the base class's version, which the source notes would be
meaningless once a real condition model exists. The radiation accessors convert between the
condition model's zero-to-one internal scale and the game's zero-to-a-hundred external one.

**Notes** — the hundred-fold scale conversion at the boundary appears here for radiation and,
in commented-out form, for health. It is the convention throughout: conditions are fractions
internally and percentages externally. A rebuild should pick one and convert at the script
and UI boundary only.

## `ef_creature_type` / `ef_weapon_type` / `ef_detector_type`

**Contract** — the world-state evaluator's classification of this creature, its weapon and
its detector. The creature type is always present; the other two fail loudly if read when
unconfigured.

**Notes** — the failure rather than a default is the decision. These feed the evaluator
patterns that decide who would win a fight, and a creature silently classified as weapon
type zero would be systematically mis-evaluated by every other creature in the level — a
bug that would show up as strange AI behaviour and never point back here.

## `create_anim_mov_ctrl` / `destroy_anim_mov_ctrl` / `OnChangeVisual`

**Contract** — bracketing hooks. Animation-driven movement takes control of the entity's
position away from the physics controller and must tell the physics support at both ends;
the create hook notifies only on the *transition into* that mode, not on a repeat. A visual
change invalidates the hit-bone area cache.

**Notes** — the "only on transition" test reads the controlled flag *before* delegating to
the base class, which is what sets it. A rebuild must preserve that read-before-set ordering
or the physics support is never notified.

## `predict_position` / `target_position`

**Contract** — both report the current position; the time argument to the predictor is
ignored. They exist as overridable hooks, and the derived creatures that move supply real
implementations.

**Notes** — worth stating rather than skipping, because a reader may assume the base class
does dead-reckoning and it does not. Anything relying on prediction must be looking at a
subclass that implements it.
