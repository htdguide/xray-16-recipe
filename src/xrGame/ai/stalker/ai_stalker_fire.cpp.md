# src/xrGame/ai/stalker/ai_stalker_fire.cpp

> Everything about a stalker and its weapon: how wide it shoots, where the round starts, whether a squadmate is in the way, which gun it wants, how it throws a grenade, and how it is crippled rather than killed.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`ai_stalker_impl.h`](ai_stalker_impl.h.md) · [`ai_stalker_space.h`](ai_stalker_space.h.md) · [`stalker_planner.h`](../../stalker_planner.h.md) · [`object_handler_planner.h`](../../object_handler_planner.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`enemy_manager.h`](../../enemy_manager.h.md) · [`visual_memory_manager.h`](../../visual_memory_manager.h.md) · [`ef_storage.h`](../../ef_storage.h.md) · [`agent_member_manager.h`](../../agent_member_manager.h.md) · [`BoneProtections.h`](../../BoneProtections.h.md) · [`trajectories.h`](../../trajectories.h.md) · [Seam: Static collision database](../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Script virtual machine](../../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: ray queries against the collision database and a ballistic solve, with per-frame caches

## Purpose

The densest file in the chapter. Six distinct mechanisms share it because all six are
consulted by the combat planner's evaluators and all six read the same weapon state.

1. **Dispersion** — how wide the stalker's cone is, from gait, stance, zoom and rank.
2. **Fire origin and direction** — where the round actually starts, which is a different
   answer in six different situations.
3. **Friendly-fire avoidance** — a five-ray probe that answers "can I hit him" and "would I
   hit one of mine" in one pass.
4. **Weapon and ammunition choice** — a four-stage search over what is held, what is
   remembered, and what can be combined.
5. **Grenades** — a minimum-velocity ballistic solve, a trajectory raycast, and a
   deliberate miss.
6. **Critical wounds** — being crippled instead of killed, and the squad vocalisations that
   follow.

## Constants

```text
DANGER_DISTANCE          = 3 metres        # radius of the danger a hit stamps on my cover
DANGER_INTERVAL          = 120 seconds     # how long that danger persists
PRECISE_DISTANCE         = 2.5 metres      # slack allowed when judging "the shot reaches"
FLOOR_DISTANCE           = 2 metres        # vertical separation above which firing is pointless
NEAR_DISTANCE            = 2.5 metres      # a probe that stops this close means "blocked"
FIRE_MAKE_SENSE_INTERVAL = 10 seconds      # how long a lost enemy is still worth shooting at
safety_fire_angle        = PI / 64         # about 2.8 degrees; the friendly-fire probe spread
```

## `GetWeaponAccuracy` — the dispersion model

**Contract** — returns the half-angle of the stalker's fire cone in radians. Pure.

```text
FUNCTION weapon_accuracy() -> real
  base = 1 degree in radians
  base = base * rank_dispersion          # veterans multiply by a smaller number

  IF the stalker is still moving along its path
    IF walking  THEN RETURN base * dispersion[walk,  stance]
    IF running  THEN RETURN base * dispersion[run,   stance]

  # stationary: the zoomed and unzoomed multipliers
  IF standing
    RETURN zoomed ? base * dispersion[stand,  stand] : base * dispersion[stand,  stand_zoom]
  ELSE
    RETURN zoomed ? base * dispersion[stand, crouch] : base * dispersion[stand, crouch_zoom]
```

**Invariants** — the base unit is one degree, so every authored dispersion number in the
configuration is read as a multiple of a degree. Rank multiplies on top of that, so a
veteran and a novice with the same weapon and stance differ by the ratio of their rank
factors and nothing else.

**Notes** — the last two branches are **swapped**: a zoomed stalker gets the *unzoomed*
multiplier and vice versa. Since aiming down a scope ought to tighten the cone and the
authored zoom multipliers are smaller than the unzoomed ones, the shipped stalker is more
accurate when *not* aiming. This is almost certainly a mistake, and it is shipped behaviour:
a rebuild that corrects it will make stalkers noticeably deadlier and will invalidate the
authored numbers. Correct it deliberately or not at all.

## `g_fireParams` — where the round starts and where it goes

**Contract** — produces the firing origin and direction. Six distinct answers, in priority
order, because the right answer depends on what the stalker is doing with its body.

```text
FUNCTION fire_params() -> (origin, direction)
  IF nothing is in hand
    RETURN (own position, world forward)              # defensive; asserts in debug

  IF the held item is a MISSILE, not a weapon
    recompute the throw if stale
    RETURN (throw origin, normalized throw velocity)  # grenades leave from the hand arc

  IF the held item is not a weapon at all
    RETURN (eye origin, eye direction)

  IF dead
    RETURN the weapon's own last firing point and direction
           # a corpse's weapon discharges along its last aim, not along its ragdoll

  IF a script animation is playing, or the animation selector has taken over
    origin = the weapon's own firing point                  # the pose is authored; trust it
    direction = sniper mode ? the head's TARGET rotation
                            : the weapon's own last direction
    RETURN

  SELECT body state
    standing AND stationary:
      RETURN (eye origin, eye direction)
    standing AND moving:
      # while moving, the eye matrix lags the animation, so fire from a synthesised
      # point: half a metre forward of the chest along the head's CURRENT rotation,
      # raised half a metre. This is what stops a running stalker shooting its own feet.
      direction = from the head's current yaw and pitch
      origin    = own centre, moved 0.5 m along direction, raised 0.5 m
    crouching:
      RETURN (eye origin, eye direction)

  # in every standing and crouching case, sniper mode overrides the direction with
  # the head's TARGET rotation rather than its current one - a sniper shoots where
  # it has decided to look, not where its head has got to
```

**Invariants** — sniper mode substitutes the *target* head rotation for the current one in
three of the six branches. That is what makes a scripted sniper hit: its shot does not wait
for the head-turn animation to converge.

## `can_kill_entity_from` — the friendly-fire probe

**Contract** — fires up to five rays from the firing origin and records two facts about the
whole bundle: whether any ray reached a living enemy, and whether any reached a living
squadmate. Results are cached per frame. Queries the collision database.

The per-ray callback accumulates *transparency*: a ray passing through glass or foliage
multiplies its remaining power by the material's transparency and continues until the power
falls below the stalker's own visual transparency threshold, at which point the stopping
distance is recorded. A ray that reaches a living entity stops there and classifies it.

```text
FUNCTION can_kill_entity_from(origin, direction, distance)
  pick_distance = 0
  probe(origin, direction)                       # 1: straight down the barrel
  IF both flags set THEN RETURN                  # nothing more to learn

  probe(origin, direction pitched DOWN by 2.8 degrees)     # 2
  IF both flags set THEN RETURN
  probe(origin, direction pitched UP   by 2.8 degrees)     # 3
  IF the friendly flag is set THEN RETURN

  remember whether an enemy was found
  probe(origin, direction yawed LEFT  by 2.8 degrees)      # 4
  restore the enemy flag                                   # see Invariants
  IF the friendly flag is set THEN RETURN
  probe(origin, direction yawed RIGHT by 2.8 degrees)      # 5
  restore the enemy flag
```

**Invariants**

- The five rays are **not symmetric in what they contribute.** The centre ray and the two
  pitch rays contribute to *both* answers. The two yaw rays contribute only to the
  friendly-fire answer: the enemy flag is saved before each and restored after. So "can I
  hit the enemy" is judged on a narrow vertical fan and "would I hit a friend" on a wider
  cross. A stalker is cautious about its flanks and does not count a flanking hit as a
  chance to kill.
- `pick_distance` — the distance at which the bundle was stopped — takes the **maximum**
  across all five rays, so it reports the most optimistic reach, and it is what
  `fire_make_sense` and the planner's line-of-fire evaluators consult.
- The whole probe is recomputed at most once per frame, stamped by frame number. Every
  public query routes through that stamp.

## `fire_make_sense`

**Contract** — the planner's "is shooting worth it" evaluator. Five gates, then a weapon-
class gate.

```text
FUNCTION fire_make_sense() -> bool
  enemy = selected enemy ; IF none THEN RETURN false

  IF pick_distance + 2.5 metres < distance to enemy  THEN RETURN false
      # the probe stopped well short of him: something solid is in the way
  IF |own height - enemy height| > 2 metres          THEN RETURN false
      # he is a floor above or below; the shot cannot matter
  IF pick_distance < 2.5 metres                      THEN RETURN false
      # the probe stopped almost at the muzzle: I am against a wall

  IF the enemy is visible right now                  THEN RETURN true

  # suppressing fire at a recently-lost enemy, but only with an automatic weapon
  last_seen = when the enemy was last visible
  IF never seen                                      THEN RETURN false
  IF now() > last_seen + 10 seconds                  THEN RETURN false
  IF there is no best weapon                         THEN RETURN false
  RETURN weapon_class IS submachine_gun OR machine_gun
```

**Invariants** — the ten-second suppression window and the automatic-weapons-only rule
together are what produce the game's characteristic blind bursts into a doorway. A pistol or
a sniper will not fire at an enemy it cannot see.

## `update_best_item_info_impl` — weapon and ammunition choice

**Contract** — chooses the best thing to kill with, in four stages, stopping at the first
that succeeds. Publishes four results: the item to kill with, the ammunition for it, and —
when neither is held — a remembered item and remembered ammunition the planner can be sent
to fetch. Cached, invalidated by taking or dropping anything and by the enemy changing.

**Stage zero — the script override.** Before anything else, if the script layer defines a
weapon-choice function, it is called with the stalker and the current best item and its
answer, if any, is adopted wholesale. This is the modding hook and it short-circuits
everything below.

```text
FUNCTION update_best_item_info()
  IF trading, or weapon selection is disabled THEN RETURN      # do not re-arm mid-trade

  IF script defines a weapon-choice function
    answer = call it(self, current best item)
    IF answer EXISTS THEN best_item = best_ammo = answer ; RETURN

  # -- stage 1: something in my own inventory that can kill ------------------
  # Scored by the "weapon effectiveness" expression when an enemy is selected -
  # a data-driven formula evaluated against (me, enemy, item) - and by plain
  # COST when there is no enemy, so an idle stalker carries its most valuable gun.
  best = none ; best_value = 0
  FOR EACH item IN own inventory WHERE item.can_kill
    value = enemy selected ? weapon_effectiveness(me, enemy, item) : item.cost
    IF value > best_value THEN best = item ; best_value = value
    ELSE IF value == best_value AND item.cost > best.cost THEN best = item
  IF best EXISTS THEN best_item = best_ammo = best ; RETURN

  # -- stage 2: a remembered item, plus something I hold ---------------------
  # Two shapes: a remembered weapon that MY ammunition would feed, or remembered
  # ammunition that would feed MY weapon.
  FOR EACH remembered item that is still useful
    IF it can kill using something in my inventory
      score it ; record it as the found item to fetch, with my item as the ammunition
    ELSE IF something in my inventory can be made killing by it
      score it ; record MY item as the weapon and it as the ammunition to fetch
  IF anything was found THEN RETURN

  # -- stage 3: two remembered items that work together ---------------------
  FOR EACH remembered item that is still useful
    IF another remembered item makes it killing
      score it ; record both as things to fetch
```

**Invariants**

- The four published results encode a *plan*, not just a choice. "I hold the weapon but must
  fetch the ammunition" and "I must fetch the weapon but hold the ammunition" are distinct
  states the planner acts on differently, and stages two and three are what distinguish them.
- Ties in effectiveness are broken by cost, which is why a stalker with two equivalent
  rifles carries the better one.

**Notes** — a disabled early-exit sits in this function, with a comment saying that it is
what made stalkers switch weapons during a firefight and that this is stupid. With it
disabled, the full search runs on every invalidation. A rebuilder should note that the
*shipped* behaviour is the expensive, re-evaluating one, and that the author considered the
cheap one worse.

## `ready_to_kill` / `ready_to_detour` / `item_to_kill` / `item_can_kill` / `remember_*`

**Contract** — the planner's evaluators over the choice above. Ready to kill requires the
best item to be the item actually in hand *and* to report itself ready. Ready to detour
additionally requires the magazine to be more than half full — a stalker will not flank on a
nearly empty gun.

## `Hit` — damage, armour and critical wounds

**Contract** — modifies an incoming hit by rank and by bone armour, then decides whether it
crippled rather than merely hurt, then records it in hit memory and passes it on.

```text
FUNCTION Hit(hit)
  power = hit.power * rank_immunity          # veterans take less

  IF bone protection exists AND the hit is a firearm hit
    armour = bone_armour(hit.bone)

    IF already wounded
      power = 1000                           # a downed stalker is finished off by anything

    ELSE IF the protection table is the third game's kind
      IF armour is non-zero
        IF hit.armor_piercing > armour
          fraction = (piercing - armour) / piercing, floored at the table's minimum
          power = power * fraction
        ELSE
          power = power * the table's minimum fraction ; do not open a wound

    ELSE IF the material library is the second game's generation
      IF piercing > 0 AND piercing > armour
        power = power * (piercing - armour) / piercing, floored at the minimum
      ELSE
        power = power * the minimum fraction ; do not open a wound

    ELSE                                      # the first game's generation
      power = max(damage - armour, damage * the minimum fraction)

  IF alive AND not already critically wounded
    IF I have a cover, the hit came from someone else, it did damage, and the
    plan permits it, stamp a DANGER LOCATION on my own cover for two minutes
       # being shot where you are standing makes that place dangerous for the squad

    IF the attacker is a living entity and I am not already down
      say "injuring" if the attacker is an enemy

    IF not wounded and not already critically wounded
      became_critical = evaluate the critical wound roll for this bone and power
      IF became_critical AND the attacker is a stalker
        tell the attacker it crippled me      # triggers its vocalisation

  IF alive AND (no hit callback OR the callback allows it)
    record the hit in hit memory, scaled by 100, or by 0 when invulnerable

  pass the modified hit to the base entity
```

**Invariants**

- **Three armour formulas coexist**, selected by which game's material library is mounted.
  All three produce a fraction of the incoming power, and all three floor it at a configured
  minimum so that armour never makes a stalker immune. A rebuild must implement all three or
  it will get the wrong damage numbers for two of the three games.
- The "do not open a wound" branch is separate from the damage reduction: a hit fully
  stopped by armour still does its floored damage but leaves no bleeding wound.
- The hit is recorded in memory *scaled by a hundred*, or by zero when invulnerable, so the
  memory's magnitudes are in a different unit from the damage system's. A rebuild that
  unifies them must re-tune every threshold that reads hit memory.
- The friendly-fire vocalisation is registered, loaded, and **never played**: the branch that
  would say it when a friend shoots you is commented out.

## `wounded`

**Contract** — the transition into and out of the downed state. Entering: notify whoever
last hit you (which registers your body with the squad's corpse manager and triggers the
attacker's "enemy down" line), destroy the character's collision capsule so the body can lie
on the ground, and unregister from the squad's combat roster. Leaving: recreate the capsule.

**Invariants** — destroying the capsule is what lets a downed stalker be a prone body rather
than a standing cylinder, and recreating it on recovery is what lets it stand again. A
rebuild must do both, in that order, or a recovered stalker will be inside the floor.

## Grenades

**Contract** — four functions, forming one mechanism.

- **`throw_target`** — records the target, invalidating the cached trajectory unless the new
  target is within ten centimetres of the old. The vertex-taking form additionally applies a
  deliberate miss.
- **`compute_throw_miss`** — displaces the aim point by a random two to five metres in a
  random direction, accepting the first of six tries that lands on navigable ground reachable
  in a straight line from the original vertex. **Stalkers do not throw grenades accurately**;
  this is what makes a thrown grenade a threat to move away from rather than a guaranteed
  kill, and the navigability check is what stops the miss landing inside a wall.
- **`update_throw_params`** — the ballistic solve. Cached against the stalker's own position
  and direction.
- **`check_throw_trajectory`** — raycasts the computed parabola against the world and clears
  the throw-enabled flag if it would hit anything before the target.

```text
FUNCTION update_throw_params()
  IF cached AND my position and direction are unchanged THEN RETURN

  # the launch point. A grenade leaves from the hand, and there are two ways to
  # find where the hand is: the missile's own authored third-person offset, or -
  # when the missile does not declare one - a fixed two metres above the feet
  # plus the missile's generic throw-point offset rotated into world space.
  origin = own position
  IF the held missile declares a third-person throw offset
    origin = origin + that offset
  ELSE
    origin.y = origin.y + 2 metres
    origin   = origin + own rotation applied to the missile's throw-point offset

  # the maximum range is a DIFFICULTY setting, not a weapon property:
  # 30 / 40 / 50 / 60 metres from easiest to hardest. Stalkers throw further at you
  # on higher difficulty.
  max_distance = { 30, 40, 50, 60 }[current difficulty]

  displacement = target - origin
  IF magnitude(displacement) > max_distance
    throw_enabled = false ; RETURN

  # solve for the launch that reaches the target with the LEAST speed - the
  # 45-degree-equivalent arc for the geometry, which maximises the time of flight
  # and so gives the player the longest warning
  time     = minimum_velocity_flight_time(displacement, gravity)
  velocity = velocity_reaching(displacement, time, gravity)

  check_throw_trajectory(time)                # raycast the parabola

  velocity = velocity * random in [0.99, 1.01]   # a final one percent of scatter
```

**Invariants**

- The minimum-velocity solve is a *gameplay* choice, not a maths convenience: it produces
  the highest, slowest arc that reaches the target, which is the most readable one and gives
  the player time to react.
- The one percent velocity jitter is applied *after* the trajectory check, so a grenade can
  clip an obstacle the check cleared. A rebuild should decide whether to jitter before or
  after; the shipped engine jitters after.
- The difficulty-indexed range table means a rebuild must expose a difficulty setting to the
  AI layer, which is otherwise unaware of one.

**Notes** — the trajectory-error helper is a stub returning true unconditionally, with a
comment demanding its deletion. It is called from nowhere.

## `zoom_state` and `update_range_fov`

**Contract** — `zoom_state` reports whether the stalker is aiming down its sights, which
feeds both the dispersion model and the perception range. It requires the stalker to be
stationary or crouched, and the weapon-handling planner to be in one of eleven operator
states — aiming, ready to aim, waiting between bursts, firing, or **reloading**. The reload
states are included deliberately, with a note saying it is to prevent field-of-view and
range switching: without them a stalker's vision would narrow and widen every time it
reloaded, and it would lose track of its enemy.

`update_range_fov` then lets the held item modify the range and field of view — which is how
a scope extends how far a stalker can see — and passes the result to the base rule.

**Invariants** — a scoped stalker literally sees further. Perception range is a function of
what it is holding and whether it is aiming, so the same stalker is a different opponent with
and without a scope.

## `too_far_to_kill_enemy`

**Contract** — **returns false unconditionally.** The distance table behind it — a pistol
gives up beyond ten metres, a shotgun beyond five, a sniper beyond seventy, everything else
beyond five — is compiled out.

The consequence is a behaviour, not an absence: **a stalker will engage at any range with
any weapon.** A pistol holder will shoot at something eighty metres away. A rebuild that
enables the table will make stalkers close distance before firing, which is a visibly
different and arguably better game — and not the shipped one.

## `undetected_anomaly` / `inside_anomaly`

**Contract** — inside-anomaly scans the stalker's touch set for a zone that restricts
movement, excluding radioactive zones (which do not constrain movement and would otherwise
make every stalker in a hot area believe it is trapped). Undetected-anomaly is that, or the
planner's belief that an anomaly is present.

## Critical wounds

**Contract** — three functions plus a squad broadcast.

- **entry conditions** — a stalker may only be crippled while standing, not mid-animation,
  holding a real weapon of an animation class that has the crippled animations (pistols,
  automatics, rifles), not inside a smart cover, and registered in the squad's combat roster.
  A stalker not in combat cannot be crippled.
- **entry** — sets the planner property and registers a callback on the global animation
  channel that will clear it when the animation ends.
- **`can_cry_enemy_is_wounded`** — decides whether a stalker may vocalise about an enemy it
  or a squadmate crippled. It requires the stalker to be *in* its combat plan, to have group
  behaviour enabled, and to be executing one of fourteen listed combat operators. The
  fourteen that permit it are the aggressive and positional ones; the eight that forbid it
  are the ones where the stalker is busy or retreating — fetching a weapon, withdrawing,
  waiting out the end of a fight, hiding from a grenade, attacking by surprise, or already
  crippled itself. A stalker sneaking up on someone does not shout.

**Invariants** — the operator list is exhaustive with no default case, so a rebuild that adds
a combat operator must extend it or the vocalisation decision becomes undefined.

## Weapon-shot effector hooks

**Contract** — on a shot, seeds the recoil effector with the stalker's own per-instance
random seed and starts it. The other three hooks are empty. The effector's *effect on aim*
is disabled; see [`ai_stalker_impl.h`](ai_stalker_impl.h.md).
