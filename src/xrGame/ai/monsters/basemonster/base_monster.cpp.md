# src/xrGame/ai/monsters/basemonster/base_monster.cpp

> The creature base's core: assembly, the two update paths, damage, death, team membership, sound pacing, and the translation from an abstract action into movement parameters.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`ai_monster_squad.h`](../ai_monster_squad.h.md) · [`state_manager.h`](../state_manager.h.md) · [`control_manager.h`](../control_manager.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`control_path_builder_base.h`](../control_path_builder_base.h.md) · [`control_direction_base.h`](../control_direction_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`monster_cover_manager.h`](../monster_cover_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`anomaly_detector.h`](../anomaly_detector.h.md) · [`anti_aim_ability.h`](../anti_aim_ability.h.md) · [`corpse_cover.h`](../corpse_cover.h.md) · [`controlled_entity.h`](../controlled_entity.h.md) · [`CharacterPhysicsSupport.h`](../../../CharacterPhysicsSupport.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`base_monster.h`](base_monster.h.md)
**Tier floor** — T2: aggregation, per-frame position correction against the navigation mesh, and damage arithmetic

## Purpose

The assembly point of every non-human creature. It builds the sub-objects listed in
[`base_monster.h`](base_monster.h.md), wires the four control channels into the control
manager, and owns the two update paths — the per-frame one and the scheduled one — that
everything else hangs off.

Four pieces of real logic live here and nowhere else: the **grouping nudge** that keeps
pack members from overlapping, the **enemy-reachability and at-home tracking** with its
hysteresis, the **damage model** for creatures with armoured skin, and the **action-to-path
translation** that turns an abstract action into a movement velocity mask.

## State

Everything owned is listed in [`base_monster.h`](base_monster.h.md). The state that is
genuinely *this file's* is the reachability hysteresis:

```text
RECORD ReachabilityTracking
  first_tick_enemy_unreachable : int   # 0 = the enemy has been reachable throughout
  last_tick_enemy_unreachable  : int
  first_tick_not_at_home       : int   # 0 = the creature has been in its territory
  last_grouping_nudge_tick     : int
```

**Invariants** — a zero in either "first tick" field is the *clean* state and is what every
query tests first. The fields are cleared on reinitialisation and whenever the condition has
been continuously false for long enough (see below).

## assembly

**Contract** — construction builds the physics support, the control manager, the four
memories with their retention periods, the enemy and corpse managers, the melee checker,
morale, the anomaly detector, the cover manager, the home object, and the four auras. It
registers the custom-ability manager as a control component and gives it the sequencer,
the triple animation and the critical-wound ability. It leaves the state manager null — the
concrete creature supplies that — and the anti-aim ability null, which is created at load
only if the section configures it.

**Invariants** — the four memory retention periods are fixed in code, not read from data:
enemies, sounds and corpses are remembered for 20 seconds and hits for 50. The hit memory's
longer window is what lets a creature go on being angry about a shot long after it has
stopped hearing anything, and it is the only asymmetry among the four.

**Notes** — the construction reaches the control manager through the object being
constructed, which is what the suppressed compiler warning in the original is about. That
is incidental: the decision is that every sub-object is handed a back-reference to its
creature at construction, so none of them needs to be found later.

## `per_frame_update`

**Contract** — runs every frame for a creature near enough to matter. Drops a consumed
corpse reference that has gone stale, runs the inherited frame update, and while alive
updates the reachability tracking, steps the footstep manager and applies the grouping
nudge. Then advances the control manager's frame pass and the physics support.

```text
FUNCTION per_frame_update()
  IF eaten_corpse is no longer a valid corpse THEN eaten_corpse = none
  inherited frame update
  IF alive THEN
    update_reachability_and_home()
    footsteps.update()
    apply_grouping_nudge()
  control.update_frame()
  physics.update_frame()
```

## `scheduled_update`

**Contract** — runs at the scheduler's rate. Updates eye-bone visibility, the anti-aim
ability, the four auras, the control manager's scheduled pass, morale, the anomaly detector
and the physics support, in that order.

**Notes** — the order is not arbitrary in one place: the anti-aim ability runs *before* the
control manager's scheduled pass, because it activates itself through the control manager
and must do so before that manager arbitrates for the tick.

## `apply_grouping_nudge`

**Contract** — nudges the creature's position by a tiny steering acceleration so that pack
members standing on each other drift apart. Runs only when the creature has a grouping
behaviour, which only exists when its section configured separation. Refuses any nudge that
would leave the navigation mesh. Writes both the physics position and the navigation vertex.

```text
FUNCTION apply_grouping_nudge()
  IF no grouping behaviour THEN RETURN

  acceleration = steering.accumulate()
  acceleration.vertical = 0                     # the nudge is strictly horizontal

  dt = seconds since the last nudge
  offset = acceleration * dt
  IF |offset| < 1e-6 THEN RETURN                # not worth the mesh queries
  IF |offset| > 0.005 THEN clamp |offset| to 0.005

  candidate = position + offset
  IF the mesh has no vertex along (current vertex, position -> candidate) THEN RETURN

  candidate = physics.virtual_move_to(candidate)   # slide along obstacles
  IF candidate is off the mesh THEN RETURN
  IF the mesh has no vertex along the corrected step THEN RETURN

  physics.set_position(candidate)
  position = candidate
  navigation_vertex = the vertex found
```

**Invariants** — the offset cap of five millimetres per nudge is the whole safety of the
mechanism. It is stated in the original as the control on how strong the separation forces
may be before the creature visibly jitters, and it is why a strong separation factor
produces a slow drift rather than a shove.

Every candidate position is validated against the navigation mesh **twice** — once for the
raw step and once after the physics slide — because the slide can move the creature
somewhere the raw step would not have gone.

**Notes** — this is a position correction applied *outside* the movement system, which is
why it must maintain the navigation vertex itself. It is the one place in the chapter where
a creature's position is written by something that is not its movement controller, and a
rebuild with a real steering integration in the movement layer deletes it.

## `update_reachability_and_home`

**Contract** — maintains the two hysteresis timers. Records when the creature first left its
home territory, and when its enemy first became unreachable, and clears each once the
condition has stopped holding.

```text
FUNCTION update_reachability_and_home()
  IF NOT home.contains(self) THEN
    IF first_tick_not_at_home = 0 THEN first_tick_not_at_home = now
  ELSE
    first_tick_not_at_home = 0

  IF no enemy THEN
    clear both enemy-unreachable timers ; RETURN

  IF enemy_is_unreachable() THEN
    IF first_tick_enemy_unreachable = 0 THEN first_tick_enemy_unreachable = now
    last_tick_enemy_unreachable = now
  ELSE IF the enemy has been reachable for 3 seconds THEN
    clear both enemy-unreachable timers
```

**Invariants** — the asymmetry is the point: becoming unreachable is recorded immediately,
becoming reachable again is only believed after three continuous seconds. Without that, a
creature whose enemy steps briefly behind a pillar flips its whole behaviour twice a second.

## `enemy_is_unreachable`

**Contract** — six independent reasons an enemy cannot be attacked, any one of which is
enough. Pure query; no state.

```text
FUNCTION enemy_is_unreachable() -> bool
  enemy_vertex_pos = mesh position of the enemy's claimed vertex
  horizontal_gap = horizontal distance from the enemy to that vertex
  vertical_gap   = |height difference| to that vertex

  RETURN true IF horizontal_gap > 0.5 AND vertical_gap > 3     # standing on something high
  RETURN true IF horizontal_gap > 1.2                          # well off any mesh cell
  RETURN true IF the enemy is outside my home territory
  RETURN true IF no point within 1.5 units of the enemy is accessible to me
  RETURN true IF the enemy's position is off the mesh entirely
  RETURN true IF the enemy's claimed vertex is invalid
  RETURN false
```

**Invariants** — the accessibility probe samples five points — the enemy's position and
four offsets of 1.5 units along the horizontal axes — and succeeds if *any* is accessible.
A single-point test fails whenever the enemy stands exactly on a restrictor boundary, which
is common enough to matter.

**Notes** — the first two tests together encode "the player has climbed onto something".
Half a unit of horizontal error plus three units of height says the enemy is on a roof or a
crate; 1.2 units of horizontal error alone says it is off the walkable surface whatever the
height. Neither number is derived. This is the mechanism behind the series' well-known
behaviour where creatures give up and circle when the player finds high ground.

Including "outside my home territory" as unreachability is a design choice, not a geometric
fact: a creature will not follow prey out of its authored area, and it reports that as
"cannot reach" rather than as a separate condition.

## `enemy_accessible` / `at_home`

**Contract** — the two questions the behaviour layer actually asks, both reading the
hysteresis timers rather than recomputing.

```text
FUNCTION enemy_accessible() -> bool
  IF first_tick_enemy_unreachable = 0 THEN RETURN true
  IF the enemy occupies the same mesh vertex as me THEN RETURN false
      # we are on the same cell and I still cannot reach it: it is above or below me
  RETURN now < first_tick_enemy_unreachable + 3000

FUNCTION at_home() -> bool
  RETURN first_tick_not_at_home = 0 OR now < first_tick_not_at_home + 4000
```

**Invariants** — both give the creature a grace period after the condition first goes bad —
three seconds for reachability, four for being at home — so that a momentary excursion does
not change its behaviour at all. The same-vertex short-circuit is the exception: sharing a
cell with an unreachable enemy is unambiguous and is answered immediately.

## `hit`

**Contract** — the creature's damage entry. Ignores collision damage when the creature has
asked to; ignores everything while invulnerable; updates the critical-wound state on the
struck bone; and applies the skin-armour model to bullet damage before passing the hit on.

```text
FUNCTION hit(h)
  IF h.type = collision AND collision damage is being ignored THEN RETURN
  IF invulnerable THEN RETURN
  IF alive AND not already critically wounded THEN
    consider_critical_wound(h.bone, h.power)

  IF h.type = bullet AND this creature has a protections section THEN
    IF skin_armour > 0 AND h.armor_piercing > skin_armour THEN
      # the round penetrated: damage scales by how far past the armour it got,
      # with a floor so a marginal penetration still hurts
      fraction = max((h.armor_piercing - skin_armour) / h.armor_piercing, hit_fraction)
      h.power = h.power * fraction
    ELSE
      # the round did not penetrate: a fraction of the damage still lands as
      # blunt force, but it leaves no wound
      h.power = h.power * hit_fraction
      h.leaves_wound = false

  inherited hit(h)
```

**Invariants** — the penetration fraction can never fall below `hit_fraction`, which is also
the non-penetration fraction. So `hit_fraction` is the single number that says "how much
damage gets through regardless", and a creature with armour is never fully immune. The
default is a tenth.

**Notes** — this model only applies to bullet damage and only when the section names a
protections section. Everything else — claws, fire, psi, anomalies — bypasses it entirely.

## `die`

**Contract** — finalises the state machine, shuts down the four auras and the anti-aim
ability, runs the inherited death, plays the death sound (a *distinct* one when the killer
is an anomaly), removes the creature from its pack, detaches the grouping behaviour, and
notifies the controlled-entity hook.

**Invariants** — the state machine's critical finalisation runs **before** the inherited
death, because a state holding a control channel must release it while the creature is still
whole.

**Notes** — removing the creature from its pack here rather than at destruction means a
corpse is no longer a pack member, so the pack's living-member count and leadership
re-election happen at the moment of death, not at cleanup. That is what makes a pack
visibly regroup when one of its members is killed.

## `change_team`

**Contract** — moves the creature between packs. Does nothing when the triple is unchanged.
Removes from the old pack, applies the change, registers with the new pack, and re-points
the grouping behaviour at the new pack. Refuses (in a checked build) to run on a dead
creature.

**Notes** — the ordering — remove, change, register — is mandatory, because both registry
calls address the pack through the creature's *current* triple.

## `set_state_sound`

**Contract** — plays one of the creature's state sounds with a delay chosen from the
settings block, scaled by how many packmates are nearby. Optionally plays immediately with
no pacing at all.

```text
FUNCTION set_state_sound(type, immediate)
  IF immediate THEN play(type) ; remember type ; RETURN

  IF type = aggressive AND the previous sound was not aggressive THEN
    play(attack_hit)          # the first roar of an attack is a different, louder sound
    remember type ; RETURN

  nearby = packmates within 20 units, plus myself
  base_delay = the settings delay for this sound type
  IF type = idle AND the player is farther than the distant-idle range THEN
    type = distant_idle ; base_delay = the distant-idle delay

  play(type, delay = base_delay * sqrt(nearby))
  remember type
```

**Invariants** — the delay scales as the **square root** of the number of nearby creatures.
That is the whole crowd-pacing rule: a pack of nine calls three times less often per member
than a lone creature, so the pack as a whole is about three times as loud rather than nine
times. Nothing derives the exponent; the effect is that a large pack is audibly a pack
without becoming a wall of noise.

The first aggressive sound of an encounter is replaced by the attack-hit sound, which is a
different, sharper cue. This is what gives the series its characteristic "the thing has
noticed you" moment, and it is driven entirely by remembering the previous sound type.

## `translate_action_to_path_params`

**Contract** — converts the abstract action the state machine selected into the movement
layer's velocity masks, and decides whether the creature should be following a path at all.
Called once per think, after the state machine has run.

```text
FUNCTION translate_action_to_path_params()
  action = the animation layer's current action
  path_enabled = true ; velocity_mask = none ; desired_mask = none

  IF action is one of { stand idle, sit idle, lie idle, eat, sleep, rest, look around } THEN
    path_enabled = false

  ELSE IF action = attack THEN
    IF attack-on-move is not enabled for this creature THEN path_enabled = false
    ELSE use the run masks (damaged variants if the creature is damaged)

  ELSE IF action is walk forward or run THEN
    use the walk or run masks, damaged variants if the creature is damaged
  ELSE IF action is one of the home walks THEN use that walk's masks
  ELSE IF action = drag THEN
    use the drag masks AND set the "moving backward" animation tag
  ELSE IF action = steal THEN use the steal masks

  IF the creature is currently invisible THEN override both masks with the invisible ones
  IF real-speed is forced THEN velocity_mask = desired_mask

  IF path_enabled THEN set both masks and enable the path ELSE disable the path
```

**Invariants** — two masks, not one. The *velocity* mask is the set of speeds the path
follower may plan with; the *desired* mask is the single speed it should actually aim for.
Forcing real speed collapses them, which is how a script that says "move at exactly this
speed" defeats the planner's freedom to slow down for corners.

The damaged variants are selected here, from a flag the memory update maintains, which is
why no state ever mentions damaged movement.

The drag action sets an animation tag as a side effect. That is the one place in this
routine that writes something other than the masks, and it is here because dragging is the
only action whose animation depends on the direction of travel.

**Notes** — the walk-backward action falls through with no masks at all, leaving both at
none while the path stays enabled. Whether that was intended is not recoverable; in
practice the action is only reached through a script.

## `get_attack_rebuild_time`

**Contract** — how long a creature may keep an attack path before rebuilding it, as a
function of the distance to its enemy: a hundred milliseconds plus twenty per unit of
distance.

**Notes** — this is a *cost* control expressed as a *quality* tradeoff. A creature far from
its enemy re-plans rarely, which is affordable and looks fine because the enemy's motion
matters less at range; a creature at close quarters re-plans every tenth of a second. The
two constants are not derived.

## `on_kill_enemy`

**Contract** — when an enemy dies: record it as a known corpse, drop every hit memory that
named it, and **clear the sound memory entirely**.

**Notes** — the sound memory is cleared wholesale rather than filtered, which throws away
sounds from other sources as well. The comment in the original says it means to remove only
the killed entity's traces. It is a real over-clear, and its effect is that a creature which
has just made a kill briefly stops reacting to everything it had heard.

## `useful` / `evaluate`

**Contract** — the creature's answer to "is this object worth going to", asked by the item
manager. An object is useful only if its position and its navigation vertex are both
accessible under the creature's restrictions, it is an alive-capable entity, and it is
dead. Every creature values every corpse equally: the evaluation is always zero.

**Notes** — the routine repairs the object's navigation vertex in passing when it is invalid
but its position is on the mesh, because the accessibility check on an invalid vertex would
fault. The original records this as a workaround for a specific defect. The *decision*
underneath is that an object's navigation vertex may legitimately be stale, and a caller
that needs it must be prepared to re-derive it from the position.

## `check_start_conditions`

**Contract** — the creature's veto on a control component activating. Delegates to the state
manager first, then adds two of its own: a rotation jump is only allowed while running at or
attacking an enemy, and a melee jump only while running, in melee, or in a running attack.

**Notes** — this is where the chapter's two layers meet. The control components do not know
about states, and the states do not know about components; this routine is the whole of
the coupling, and it is deliberately tiny.

## `on_network_event`

**Contract** — handles four entity events: taking or buying an item (place it in the
creature's pack and attach it), selling or dropping one (release it and deny touch
notifications for two seconds so it is not immediately re-detected), and being told that
something was killed (route it into `on_kill_enemy`).

**Notes** — creatures have inventories at all only because the games' trading and corpse
systems are uniform. The two-second touch denial after a drop is the generic fix for "an
object I just put down is a new object near me".

## `restrictions_changed`

**Contract** — when the set of volumes the creature may move in changes, the state manager
is reinitialised. That is a heavy response to a light event, and it is correct: a plan built
against the old restrictions may now be invalid at any point along it, and there is no
cheaper way to know.

## `load_effector`

**Contract** — reads a full post-process-and-camera attack effector description from a named
section: the duality, grey, blur and noise parameters, three colour triples, the three
post-process timings and the four camera-shake parameters. Requires the noise rate to be
non-zero.

## `play_particles`

**Contract** — creates a particle effect oriented along a direction at a position, with an
orthonormal basis generated around the direction. Either sets the effect's transform
outright or parents it to that transform, depending on whether it should follow the creature.

## `update_eyes_visibility`

**Contract** — hides the creature's eye bones when it occupies too little of the screen,
and shows them when it is close enough or dead. Does nothing for a creature whose section
does not name eye bones. Forces an immediate bone recomputation on the transition from
hidden to visible.

```text
FUNCTION update_eyes_visibility()
  IF no eye bones named THEN RETURN
  visible = (not alive) OR screen_coverage() > 0.05
  set both eye bones' visibility to `visible`
  IF they were hidden and are now visible THEN recompute the pose immediately
```

**Notes** — eyes are separate bones with their own geometry, and at distance they are a few
pixels of expensive detail. The threshold of 0.05 is in normalised screen units. The forced
recomputation exists because a bone made visible between two pose evaluations would
otherwise render with a stale transform for one frame — visible as an eye in the wrong
place. Dead creatures always show their eyes, because a corpse is something the player
looks at closely.

## `screen_coverage`

**Contract** — the geometric mean of the creature's bounding box's width and height in
normalised screen space, computed by projecting the box's eight corners with the full
view-projection transform and taking the extent of the result.

**Notes** — the geometric mean rather than the diagonal or the area gives a measure that is
linear in apparent size for any aspect ratio, which is what a threshold on "is this worth
detail" wants.

## creation of the control channels

**Contract** — the four base controllers are constructed, registered with the control
manager under both their base identity and as the default owner of their channel, and the
path builder is additionally installed as the creature's movement manager. A concrete
creature overrides the construction to substitute its own controllers.

**Invariants** — a channel always has a base controller, so an ability that releases a
channel returns it to something rather than to nothing. That is what makes the capture and
release discipline safe.

## `drop_references`

**Contract** — clears a destroyed object out of the state manager, the custom-ability
manager, all four memories, the enemy and corpse managers, the pack registry and the physics
support, then re-derives the creature's memory summary. The corpse memory is cleared even
for a dead creature, because a corpse's memory is still read by the things eating it.

## attack-on-move accessors and aura queries

**Contract** — each returns one tuned parameter, routed through a debug override so a
developer can change it live by name. The far radius is additionally clamped to 100 units.
The three aura queries return each aura's current influence at the player, and the detector
sound trigger plays each aura's proximity cue.

**Notes** — the debug override is a lookup in a global variable table keyed by the
creature's class name and the parameter name. In a release build it compiles to the value
itself. It is incidental machinery, but it records something load-bearing: every one of
these parameters was tuned by hand at runtime.

## `check_eaten_corpse_draggable`

**Contract** — can the corpse the creature is eating be dragged? Answered by asking the
corpse's *model* whether its embedded configuration declares a set of bones a creature may
grab it by. A corpse with no such declaration cannot be dragged.

**Notes** — the capability lives in the model data, not in the creature and not in the
corpse's configuration section. That is how a model can be authored as grabbable without any
code knowing which models those are.
