# src/xrGame/ai/monsters/bloodsucker/bloodsucker.cpp

> The bloodsucker: how it becomes invisible, when it lets itself be seen, what it wants from the player, and how it takes his camera.

**Needs** — [`bloodsucker.h`](bloodsucker.h.md) · [`bloodsucker_state_manager.h`](bloodsucker_state_manager.h.md) · [`bloodsucker_vampire_effector.h`](bloodsucker_vampire_effector.h.md) · [`ai_monster_bones.h`](../ai_monster_bones.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`control_rotation_jump.h`](../control_rotation_jump.h.md) · [`Actor.h`](../../../Actor.h.md) · [`ActorCondition.h`](../../../ActorCondition.h.md) · [`material_manager.h`](../../../material_manager.h.md) · [`CharacterPhysicsSupport.h`](../../../CharacterPhysicsSupport.h.md)
**Used by** — [`bloodsucker.h`](bloodsucker.h.md)
**Tier floor** — T2: a visual swap with a damage-model reload, plus per-frame want accumulation

## Purpose

The bloodsucker is the chapter's most elaborate creature and the one whose abilities reach
furthest outside itself. Everything in this file is one of four mechanisms: **invisibility**
(a second model, not a shader trick), **the visibility ladder** (when the creature chooses to
be seen), **the vampire drive** (what it wants and when it is allowed to take it), and
**alien control** (taking the player's camera).

Its ordinary behaviour — approach, melee, flee, rest — is entirely the shared machinery. What
makes it feel different is that all of that runs while the creature is invisible.

## State

```text
RECORD Bloodsucker extends Monster
  # invisibility
  default_visual, predator_visual : text    # two models; the second is the invisible one
  is_predator          : bool               # which model is currently worn
  forced_vis_state     : int                # -1 unset, +1 forced visible, -1 forced invisible
  invisible_particle   : text
  invisible_velocity   : (linear, angular)

  # visibility ladder
  visibility           : {none, partial, full}
  forced_visibility    : optional<same>     # a script override
  full_visibility_radius    : real = 5
  partial_visibility_radius : real = 10
  change_min_delay     : int = 1000 ms
  last_change_time     : int
  runaway_started      : int                # 0 = not running away

  # vampire
  want_value           : real [0..1]        # fills continuously
  want_speed           : real               # per second, from configuration
  min_delay            : int                # per-creature cooldown
  LAST_VAMPIRE_TIME    : int   # SHARED BY EVERY BLOODSUCKER IN THE PROCESS
  wound, gain_health, distance : real
  hits_before_vampire  : int
  sufficient_hits      : int + a per-creature random offset of -1, 0 or +1
  vampire_effector     : PostProcessInfo
  vampire_animation    : TripleAnimationConfig

  # critical hits
  critical_hit_chance  : real = 0.25
  last_critical_tick   : int

  # bones and grab
  spine_bone, head_bone : bone references
  bone_channels         : BoneSet
  collision_off, collision_hit_off : bool
  grab_target, grab_bone, jump_position, jump_factor : the scripted drag-jump payload
```

**Invariants** — the shared last-vampire timestamp is a process-wide value, not per creature.
Two bloodsuckers cannot grab the player in quick succession, however far apart they are. That
is a game-feel decision made as a static, and it is the single most surprising line in the
creature.

`is_predator` means "currently wearing the invisible model", and the forced visibility state
overrides it in both directions with an explicit tri-state, which is why the transitions read
as three-way rather than boolean.

## configuration

Beyond the base creature's keys, the bloodsucker reads:
`Velocity_Invisible_Linear` / `Velocity_Invisible_Angular`, `vampire_effector` (a section
name), `Vampire_Delay`, `Vampire_Want_Speed`, `Vampire_Wound`, `Vampire_GainHealth`
(default 0.5), `Vampire_Distance` (default 1), `Vampire_Sufficient_Hits` (default 5),
`Predator_Visual`, `Particle_Invisible`, `critical_hit_chance` (default 0.25),
`visibility_state_change_min_delay` (default 1000 ms), `full_visibility_radius` (default 5),
`partial_visibility_radius` (default 10), plus the seven sound keys, plus two presence flags:
`is_friendly` (suppresses the running attack) and `is_no_fx` (suppresses the damage-reaction
effects), and `collision_hit_off` (the creature takes no collision damage at all).

Enemy memory is extended to **40 seconds**, twice the base. A bloodsucker that has seen you
keeps hunting long after anything else would have given up.

## `Load`

**Contract** — registers the creature's abilities with the custom-ability manager (running
attack unless friendly, rotation jump, jump), registers about twenty animations against the
velocity table, registers the transition table and the action links, then reads the
bloodsucker-specific settings and calls the base's post-load.

**The animation registration** is done twice in full, once with damage-reaction effects
attached and once without, selected by the presence of `is_no_fx`. The two lists are
identical otherwise. A rebuild registers once and attaches the effects conditionally.

**The substitution table** — five entries: damaged run, damaged walk and damaged stand
replace their normal forms when the damaged flag is set, and the run animation is replaced by
its turning variants when the run-turn flags are set. Those five lines are the whole reason
a wounded bloodsucker moves differently, and no state mentions it.

**The acceleration chains** — walking forward accelerates into running (and into either
running turn), and damaged walking into damaged running. That is how a creature that is told
to run from a standing start plays a transition rather than snapping into a run cycle.

## visibility

The three-state ladder is the bloodsucker's signature and the whole of its stealth.

```text
FUNCTION update_visibility()
  IF dead THEN                                            want full
  ELSE IF within (runaway_started + 500 ms) THEN          want partial
  ELSE IF within (runaway_started + 3000 ms) THEN         want none
  ELSE IF there is an enemy THEN
    d = distance to the enemy
    IF d <= full_radius     THEN want full
    ELSE IF d <= partial_radius THEN want partial
    ELSE                             want none
  ELSE                                                    want full
```

**Invariants** — the ladder is driven by *distance to the enemy*, not by the creature's own
intent. A bloodsucker becomes visible because it has got close, not because it decided to.
That is what makes the encounter read the way it does: the shimmer resolves into a creature
as it closes.

The runaway window is the exception and runs in the opposite direction: a creature that has
just broken off is *briefly partial*, then fully invisible for the rest of three seconds,
regardless of range. The half-second of partial visibility at the start is the player's one
chance to see which way it went.

A dead bloodsucker is always fully visible.

## `set_visibility_state`

**Contract** — applies a wanted state, subject to the forced override and a minimum dwell
time. Swaps the model when crossing into or out of full visibility, and plays the visibility
sound when going fully invisible.

```text
FUNCTION set_visibility_state(wanted)
  IF a forced state is set THEN wanted = the forced state
  IF wanted is unset OR wanted = current THEN RETURN
  IF now < last_change_time + change_min_delay THEN RETURN   # rate limit

  last_change_time = now ; current = wanted
  IF wanted = full    THEN stop_being_predator()
  ELSE IF wanted = partial THEN start_being_predator()
  ELSE                          play(visibility_change sound)
```

**Invariants** — the dwell time is what stops the creature flickering when the player circles
at exactly the radius. It applies to *every* change including the forced one, which means a
script forcing a state immediately after another change is silently ignored for up to a
second.

**Notes** — the *none* branch changes no model: partial and none wear the same predator
visual, and the difference between them is purely that the creature is not drawn at all in
*none*. So the three states are really two models and a render suppression.

## `start_being_predator` / `stop_being_predator`

**Contract** — swaps the creature's visual between the two models, **reloads the damage
model from the section**, restarts the animation layer, emits the invisibility particle
effect and plays the change sound. Each is a no-op if already in that state.

**Invariants** — the damage model is reloaded on every swap. That is not decoration: the two
models have different skeletons, so the bone-to-damage-multiplier map must be rebuilt or hits
land on the wrong body parts. This is the load-bearing consequence of implementing
invisibility as a model swap.

The animation layer is restarted for the same reason — the motion identifiers are per model.

**Notes** — both routines begin with a three-way dance over the forced visibility flag that
is hard to read and amounts to: a forced state wins, and within a forced state the swap is
skipped. The physics support is told about the visual change only in the *stop* direction,
not the *start* direction. Whether that asymmetry is deliberate is not recoverable; the
likely consequence is that the collision shell keeps the visible model's proportions while
invisible.

## the vampire drive

**Contract** — three pieces. The **want value** fills continuously at a configured rate every
frame the creature is alive and saturates at one. `WantVampire` is true only at saturation.
`SatisfyVampire` zeroes it and adds the configured health to the creature, clamped to its
maximum.

**Invariants** — the want fills on the *per-frame* path, not the scheduled one, so it fills
in real time rather than at the creature's think rate. A distant bloodsucker still gets
hungry.

**The second gate** is `done_enough_hits_before_vampire`: the creature must have landed at
least `sufficient_hits` running-attack blows, offset by a per-creature random of −1, 0 or +1
chosen once at load. That offset is what stops a pack of bloodsuckers all grabbing on the
same blow count.

**The third gate** is the shared cooldown described under State.

## `activate_vampire_effector`

**Contract** — starts the two six-second effectors on the player: a camera effector that
pulls the view toward the creature's head, and a post-process wash. Both are described in
[`bloodsucker_vampire_effector.cpp`](bloodsucker_vampire_effector.cpp.md). Six seconds is a
literal here and matches the grab's duration.

## `HitEntity`

**Contract** — the creature's melee, with a critical roll. A critical multiplies the impulse
by ten; the damage is unchanged. Delegates the rest to the base.

**Notes** — a critical hit is *only* an impulse, so it throws the player rather than hurting
him more. Combined with the attack state's rule that a critical is followed by an immediate
break-off, the effect is the creature's characteristic hit-and-vanish.

The critical timestamp is recorded by the attack state, not here; this routine only rolls.

## `check_start_conditions`

**Contract** — the creature's ability veto. **Never jumps** — the generic jump ability is
refused outright even though it is registered. Refuses the running attack while invisible.
Everything else defers to the base.

**Notes** — registering the jump ability and then refusing it unconditionally is dead
configuration. The scripted jump path goes through a different route and is unaffected.

## `CheckSpecParams`

**Contract** — the hook by which an animation tag selects a special animation. Three tags are
handled: checking a corpse runs the corpse-check animation through the sequencer; threatening
and standing scared each set an animation directly and return.

## bone control

**Contract** — at spawn, the spine and head bones are resolved and given a callback into the
additive bone layer, then registered as four rotation channels (two axes each). The callback
is only installed when the creature has **no physics shell**, because a ragdolled creature's
bones are driven by the solver and a second callback would fight it.

**Notes** — the routine that would actually *use* those channels — turning the head and spine
toward a sound — is present but entirely commented out. So a shipping bloodsucker registers
four bone channels and never commands them. The disabled code is the clearest statement in
the chapter of what the bone layer was for: turn the head within a maximum angle, and only
turn the torso when the head alone cannot reach.

## alien control

**Contract** — a switch that activates or deactivates the camera takeover described in
[`bloodsucker_alien.cpp`](bloodsucker_alien.cpp.md). While active, the creature plays its
alien drone every scheduled update.

## the drag jump

**Contract** — a scripted sequence: capture an entity by a named bone, play a linking
animation, and on its completion go invisible and leap to a stored position.

```text
FUNCTION set_drag_jump(target, bone_name, landing, factor)
  remember all four ; drag_pending = true ; animating = true

FUNCTION start_drag()
  IF animating THEN
    seize the animation channel from script
    play the link animation once, with a completion callback
    animating = false

ON link animation complete:
  go invisible ; scripted_jump(landing, factor)      # and roar
```

**Invariants** — the two flags sequence three ticks of the state manager: the manager routes
to its custom state while either flag is set, the state captures the target, and the
animation's completion drives the leap. See
[`bloodsucker_state_manager.cpp`](bloodsucker_state_manager.cpp.md).

**Notes** — the linking animation is named as a string literal in code
(`boloto_attack_link_bone`), as is the jump data set loaded at reinitialisation. Those names
belong to one specific scripted encounter, and a creature without those motions in its model
cannot be given a drag jump. This is the one place in the chapter where a *level's* content
is named from the engine.

## render suppression

**Contract** — the creature draws nothing at all while its visibility state is *none*. Note
that this tests the raw state, not the forced-override accessor, so a script forcing full
visibility still fails to draw the creature until the underlying state catches up.

## `manual_activate` / `manual_deactivate`

**Contract** — a second, cruder invisibility: mark the creature invisible and stop rendering
it, with no model swap and no sound. Used by the alien control, which needs the creature gone
rather than shimmering.

## the disabled and the dead

The commented-out post-state hook would have set the aggression flag from the current state;
the base derives it from perception instead. The `LookDirection` routine is entirely
commented out, as described under bone control. Two of the three critical-hit tuning
constants are commented out in the namespace that declares them.
