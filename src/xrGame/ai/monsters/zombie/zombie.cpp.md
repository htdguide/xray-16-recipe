# src/xrGame/ai/monsters/zombie/zombie.cpp

> Implements the zombie, whose one idea is a feigned death that is cheaper to survive each time until it runs out.

**Needs** — [`zombie.h`](zombie.h.md) · [`zombie_state_manager.h`](zombie_state_manager.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`anim_triple.h`](../anim_triple.h.md) · [`ai_monster_bones.h`](../ai_monster_bones.h.md)
**Used by** — [`zombie.h`](zombie.h.md)
**Tier floor** — T3: an animation table plus a health-band rule

## Purpose

Three decisions define the zombie and a rebuild must reproduce all three.

**Feigned death is triggered by damage, not chosen by the mind.** It lives in the hit
handler and bypasses the behaviour tree completely; the behaviour tree is then suppressed
for its duration by a single check at the top of the selection function. A rebuild that
tries to model it as a state will have to solve the interruption problem the original
sidesteps.

**Each feign costs more health than the last.** The trigger is not "health below the
threshold"; it is "health below a band that narrows every time you use one". With `n`
feigned deaths granted and `k` already used, the zombie feigns only when its health falls
below `threshold * (1 - k/n)`. The first is nearly free, the last needs the zombie almost
dead, and once all are used the bar sits at zero and the zombie can only actually die.

**Only bullets trigger it, and at most once a frame.** A shotgun blast that registers as
several hits in one frame counts once, which is what stops a single burst from burning
every feign at once.

## `Load`

**Contract** — base creature load, then the animation table, then post-load. Reads two
values from the creature's configuration section.

```text
FUNCTION Load(section)
  base.Load(section)
  animation.load_acceleration_parameters(section)
  animation.acceleration_chain(from = walk_forward, to = run)

  # rolled per individual: a zombie gets between 1 and the configured count.
  fake_death_count       = 1 + random_below(config.read_int(section, "FakeDeathCount"))
  health_death_threshold = config.read_real(section, "StartFakeDeathHealthThreshold")

  # seven motions, each with a four-way directional effect set attached
  # (forward / back / left / right), which is what makes a hit visibly stagger
  # the zombie in the direction it came from
  effects = { "fx_stand_f", "fx_stand_b", "fx_stand_l", "fx_stand_r" }
  declare(stand_idle,       "stand_idle_",      velocity = idle, posture = standing, effects)
  declare(stand_turn_left,  "stand_turn_ls_",   velocity = turn, ... )
  declare(stand_turn_right, "stand_turn_rs_",   velocity = turn, ... )
  declare(walk_forward,     "stand_walk_fwd_",  velocity = walk, ... )
  declare(run,              "stand_run_",       velocity = run,  ... )
  declare(attack,           "stand_attack_",    velocity = turn, ... )
  declare(die,              "stand_die_",       velocity = idle, ... )   # plays once, not looped

  link(stand_idle | sit_idle | lie_idle | eat | sleep | rest | drag
       | look_around                -> stand_idle)
  link(walk_forward | walk_backward -> walk_forward)
  link(steal                        -> walk_forward)
  link(run                          -> run)
  link(attack                       -> attack)

  base.post_load(section)
```

**Notes** — the zombie declares an explicit death motion and maps *stealing* onto walking
rather than onto idle, which is the only place its table differs in kind from the
tushkano's. Damaged-gait variants are present but commented out, as for every creature in
this family.

## `reload`

**Contract** — attaches the four feigned-death animation triples, each a named
wind-up / hold / recovery triple: fall, lie there, get up. All four capture the animation
channel and none loops its middle phase, so a feigned death is a fixed-length sequence that
nothing else can interrupt. Their names are literals in code, not configuration: a model
must supply exactly these twelve motions.

## `Hit` — where feigned death is decided

**Contract** — applies the base creature's hit handling first, so damage lands regardless.
Then, on a living zombie only, decides whether this hit triggers a feign.

```text
FUNCTION Hit(hit)
  base.Hit(hit)
  IF NOT alive THEN RETURN

  IF hit.type == firearm AND current_frame != last_hit_frame
    IF no animation triple is already running
       AND now() > time_resurrect + 2000 milliseconds      # cannot re-feign straight away
       AND health < health_death_threshold
      # the narrowing band: each feign already spent raises the bar
      used = fake_death_count - fake_death_left
      band = health_death_threshold - used * health_death_threshold / fake_death_count
      IF health < band
        active_variant  = random one of the four
        start the chosen animation triple
        movement.stop()
        time_dead_start = now()

        # clamp so the counter never goes negative; once it reaches zero and stays
        # there, `used` equals the full count and the band above evaluates to zero,
        # which no living zombie's health can be below. That is how the mechanic
        # runs out — not by an explicit check.
        IF fake_death_left == 0 THEN fake_death_left = 1
        fake_death_left = fake_death_left - 1

  last_hit_frame = current_frame
```

**Invariants**

- The damage is applied *before* the feign decision, so a hit that kills outright kills; the
  feign is not a save.
- The two-second lock-out after a recovery is what stops a zombie from feigning again the
  instant it stands up, which would otherwise be the dominant strategy against continuous
  fire.
- The counter clamp is not defensive coding; it is the exhaustion mechanism. A rebuild that
  replaces it with an explicit "no feigns left" test must also make the band arithmetic
  agree, or the last feign will behave differently.

## `shedule_Update` — where a feigned death ends

**Contract** — when a feign is running and five seconds have passed since it began, asks the
animation triple to leave its hold phase and play its recovery, clears the feign timestamp,
and stamps the recovery time for the lock-out. Five seconds is hard-coded.

## `fake_death_fall_down` / `fake_death_stand_up`

**Contract** — the same mechanic exposed for a caller to drive: fall down refuses if any
animation triple is already running, otherwise picks a random variant, starts it and stops
the creature; stand up searches the four variants for the running one and asks it to advance
to recovery. Neither touches the counters or the health band, so a caller-driven feign is
free and unlimited.

## `vfAssignBones` / `BoneCallback`

**Contract** — resolves the head and spine bone handles by name and configures a five-axis
bone-rotation rig over them: three axes on the spine, two on the head.

**This rig never runs.** The two lines that would install the callback on those bones are
commented out, with a note that callbacks must not be set when a physics shell exists. The
bone-update function is therefore never called, and the rig's configuration decides nothing.
A zombie's head and spine do not track anything. A rebuilder should either wire the rig up
deliberately — and then solve the physics-shell conflict the comment warns about — or drop
it; what must not happen is describing the shipped zombie as having a look-at rig.

## `net_Spawn` / `reinit`

**Contract** — spawn resolves the bones after the base spawn succeeds. Re-initialisation
resets the bone rig, clears the feign timestamps, restores the feign counter to the
per-individual roll, and clears the active variant. Note that the *roll itself* happens at
load, not at re-initialisation, so a zombie restored from a save keeps the number of feigned
deaths it was born with.

## Notes

**A debug key binding** lets a developer force a fall and a stand-up. It is compiled only in
debug builds and decides nothing about shipped behaviour.
