# src/xrGame/ai/stalker/ai_stalker.cpp

> The stalker's lifecycle and its update loop: two brains per tick, thirty vocalisations and a hundred-odd fire-queue numbers read from configuration, and a rank that scales three things at once.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`ai_stalker_impl.h`](ai_stalker_impl.h.md) · [`ai_stalker_space.h`](ai_stalker_space.h.md) · [`stalker_planner.h`](../../stalker_planner.h.md) · [`stalker_movement_manager_smart_cover.h`](../../stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](../../sight_manager.h.md) · [`stalker_animation_manager.h`](../../stalker_animation_manager.h.md) · [`object_handler_planner.h`](../../object_handler_planner.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`agent_manager.h`](../../agent_manager.h.md) · [`cover_evaluators.h`](../../cover_evaluators.h.md) · [Seam: Threads, atomics and process services](../../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`ai_stalker.h`](ai_stalker.h.md)
**Tier floor** — T2: owned sub-managers, a configuration read, and two nested update calls

## Purpose

Four things live here and each is load-bearing.

**The per-tick shape.** A stalker runs *two* independent planners: the goal/plan/action
planner that decides what to do, and a separate weapon-handling planner that decides what
the hands are doing with whatever item is held. They run on different clocks — the first on
the scheduled update, the second on the frame update — and neither knows about the other.
That separation is why a stalker can reload while running to cover.

**Rank scales three things, linearly, once.** A stalker's rank from zero to a hundred is
converted at spawn into three multipliers — damage taken, how far it can see, and how wide
its shots scatter. Immunity and visibility interpolate from a novice value to an experienced
one; dispersion interpolates the other way, so a veteran is more accurate. All six endpoints
are authored in one shared configuration section, so rank tuning is global rather than
per-character.

**Fire behaviour is almost entirely data.** Five weapon classes times three range bands
times four numbers — burst length minimum and maximum, and the pause between bursts, minimum
and maximum — plus two range-band boundaries per class. A hundred and ten numbers, every one
of them readable from configuration with a code default. This is the biggest single block of
authored AI tuning in the game and it is what makes a shotgunner behave unlike a sniper.

**Voice is a mask algebra, not a priority number.** Thirty named vocalisations are
registered, each with a *mask* that says which other vocalisations it may interrupt and be
interrupted by. See [`ai_stalker_space.h`](ai_stalker_space.h.md) for the algebra.

## `reinit` — re-initialisation

**Contract** — resets every cache and sub-manager to a known state. Runs on spawn and on
restore. Allocates the six cover evaluators.

```text
FUNCTION reinit()
  weapon_handling_planner.reinit() ; sight.reinit() ; base.reinit() ; animation.reinit()

  # the voice bank is per-character, not per-class: two stalkers of the same
  # section speak with different voices because the character record names a prefix
  sound.set_prefix(character_record.voice_prefix)
  load_sounds(section)

  physics.init()
  invalidate the item cache and the sale cache

  # six cover evaluators, each with an INERTIA in milliseconds: the minimum time
  # a chosen cover must be kept before a better one may displace it. These are the
  # numbers that decide whether a firefight looks jittery or committed.
  close_to_enemy  : inertia 3000
  far_from_enemy  : inertia 3000
  best            : inertia 1000
  angle           : inertia 5000
  safe            : inertia 1000
  ambush          : inertia 3000

  friendly-fire cache cleared ; cover cache cleared
  weapon_shot_seed = server time        # per-stalker recoil pattern

  throw cache cleared, throw enabled, throw interval = 20 seconds
  planner.set_property(critically_wounded, false)

  # per-body-part critical wound weights, parsed from a comma-separated string
  # in the CHARACTER record rather than the class section — so two stalkers of
  # one class can differ in where a critical wound is likely to land
  critical_wound_weights = parse_reals(character_record.critical_wound_weights)
```

**Invariants** — the cover evaluators are destroyed in `net_Destroy` and recreated here, so
a stalker that is re-initialised without being destroyed leaks them. In practice
re-initialisation follows construction or destruction.

## `LoadSounds`

**Contract** — registers thirty named vocalisations from the configuration section. Each
registration carries: the sound resource name, a priority number, a perception type (dying,
injuring or talking — this is what creature *hearing* classifies the sound as), a *group*
number, an interruption mask, the identifier callers use, the bone the sound emits from, and
an optional data visitor that lets the sound carry AI-perception attributes.

Five of the thirty have a fallback: when the configuration section does not declare
`sound_wounded`, `sound_enemy_lost_no_allies`, `sound_enemy_lost_with_allies` or
`sound_throw_grenade`, an existing sound is registered under the missing identifier instead.
That is what lets an older game's configuration — which has fewer sounds — load unmodified.
A rebuild must keep these fallbacks or the earlier two games will fail to spawn stalkers.

**Notes** — one fallback is wrong and audibly so: the grenade-throw vocalisation is
registered with the interruption mask belonging to *killing a wounded enemy*, in both the
present and the absent branch. A stalker announcing a grenade therefore competes for the
voice channel against the wrong set of lines.

## `reload` — the configuration read

**Contract** — reloads every sub-manager from the section, then reads the stalker's own
tuned numbers. Runs once per spawn, after `Load`.

```text
FUNCTION reload(section)
  planner.setup(self)
  base.reload(section) ; step_manager.reload(section)
  weapon_handling.reload(section) ; sight.reload(section) ; movement.reload(section)

  # eight dispersion multipliers, applied on top of one degree and the rank factor
  dispersion[walk,  stand]  = read(section, "disp_walk_stand")
  dispersion[walk,  crouch] = read(section, "disp_walk_crouch")
  dispersion[run,   stand]  = read(section, "disp_run_stand")
  dispersion[run,   crouch] = read(section, "disp_run_crouch")
  dispersion[still, stand]  = read(section, "disp_stand_stand")
  dispersion[still, crouch] = read(section, "disp_stand_crouch")
  dispersion[still, stand, zoomed]  = read(section, "disp_stand_stand_zoom")
  dispersion[still, crouch, zoomed] = read(section, "disp_stand_crouch_zoom")

  queue_section = section            # see Notes: the override never takes effect

  # five weapon classes: pistol, shotgun, sniper, machine gun, and a generic
  # automatic class. Each has three range bands with four numbers, every one
  # optional with a code default.
  FOR EACH class IN { pistol, shotgun, sniper, machine_gun, automatic }
    FOR EACH band IN { close, medium, far }
      min_size, max_size, min_interval, max_interval
        = read_or_default(queue_section, class, band)
    medium_boundary = read_or_default(queue_section, class, "queue_fire_dist_med", 15 metres)
    far_boundary    = read_or_default(queue_section, class, "queue_fire_dist_far", 30 metres)

  power_fx_factor = read(section, "power_fx_factor")
```

The defaults encode the intended character of each weapon class, and are worth stating
because most configuration files do not override them:

| Class | Close | Medium | Far |
|---|---|---|---|
| pistol | 3–5 rounds every 0.5–0.75 s | 2–4 every 0.75–1 s | 1 every 1–1.25 s |
| shotgun | 1 every 0.5–1 s | 1 every 0.75–1.25 s | 1 every 1.25–1.5 s |
| sniper | 1 every 3–4 s | 1 every 3–4 s | 1 every 3–4 s |
| machine gun | 4–10 every 0.3–0.5 s | 4–6 every 0.5–0.75 s | 1–6 every 0.5–1 s |
| automatic | inherits the generic `weapon_*` values, same shape as the machine gun |

Band boundaries default to 15 and 30 metres for every class.

**Notes** — a per-weapon **fire-queue section override** is read from
`fire_queue_section` and then immediately discarded: the test that was meant to accept a
non-empty, existing section instead rejects every non-empty string, so the queue section is
always the creature's own section. The override is dead. A rebuilder implementing this
feature should know that the shipped games behave as if it did not exist, and that
"restoring" it would change fire behaviour wherever a configuration file sets it.

## `net_Spawn`

**Contract** — requires a human-stalker spawn record. Reads group behaviour from the spawn
flags, spawns the weapon-handling side and the base entity side, sets money, reloads
animations, seeds the body and head orientation from the spawn record's torso yaw, adopts
the spawn record's graph vertex and — if it is reachable under the stalker's restrictions —
its destination graph vertex. Asserts that the game graph, level graph and cross table all
exist and agree. Loads per-model immunities and bone protection from the *visual's* embedded
configuration, not from the class section. Computes the three rank multipliers. Adopts a
panic threshold from the character record when one is set. Aims the sight at the current
facing and runs one look step so the stalker is not facing nowhere on its first frame.

```text
  rank   = clamp(character_rank, 0, 100) / 100
  immunity   = novice_immunity   + (experienced_immunity   - novice_immunity)   * rank
  visibility = novice_visibility + (experienced_visibility - novice_visibility) * rank
  dispersion = experienced_disp  + (novice_disp - experienced_disp) * (1 - rank)
```

**Invariants** — immunities and bone protection come from the *model*, so re-skinning a
stalker changes how much damage it takes. That is the mechanism armour uses, and it is why
the same class section produces very different durability across characters.

## `UpdateCL` — the frame update

**Contract** — runs the weapon-handling planner, the physics, the sight, the step manager
and the recoil effector, once per rendered frame while alive. Profiled throughout.

```text
FUNCTION frame_update()
  IF alive
    IF multithreaded object handling is enabled AND the weapon planner is initialised
      queue the weapon-handling update onto the frame's parallel work list
    ELSE
      run the weapon-handling update inline

    # the "moving in danger" vocalisation, raised from movement state alone
    IF moving AND mental state is danger AND standing AND running
      say running_in_danger

  base.frame_update()
  physics.frame_update()

  IF alive
    # the sight update can throw out of script; on any failure the stalker falls
    # back to "look where you are already looking", which is always valid, and
    # retries. This is the only place in the stalker where an exception is used
    # as control flow, and it exists because sight targets come from Lua.
    TRY sight.update() CATCH sight.setup(current_direction) ; sight.update()

    execute the look step for this frame's delta
    step_manager.update()
    IF the recoil effector is active THEN advance it
```

**Notes** — the weapon-handling update is wrapped in a catch-all that, on any failure,
resets the weapon goal to idle and runs it again. Weapon goals come from Lua and a bad goal
must not kill the frame. A rebuild without exceptions needs an equivalent: a weapon planner
that reports failure and a caller that falls back to idle.

**A walking-in-danger vocalisation exists in the enumeration and is never played.** The
branch that would have played it is commented out, and its registration is commented out in
the sound loader. Only the running variant is live.

## `shedule_Update` — the scheduled update

**Contract** — the stalker's coarse tick. Runs vision, memory and the goal/plan/action
planner. Its cadence degrades with distance and load, which is what makes a hundred stalkers
affordable.

```text
FUNCTION scheduled_update(delta)
  IF the weapon planner has not been initialised yet THEN run it once here
     # so that a stalker has hands before it has a first frame

  discard network updates older than the interpolation horizon

  IF alive
    run any animation callbacks deferred from the frame update
    agent_manager.update()          # the squad's shared picture
    run vision
    process_enemies()               # borrow a squadmate's enemy; see ai_stalker_misc.cpp
    memory.update(delta)

  base.scheduled_update(delta)

  IF this machine is the authority
    IF a script has control THEN run the Lua action queue ELSE Think()
    record the update time
    update touch perception
    push a network snapshot of position, body yaw and head rotation

  inventory_owner.update(delta)
  physics.scheduled_update(delta)
```

**Notes** — the multithreaded vision path is present and **disabled by a hard false** beside
the flag that would have enabled it. Vision always runs inline. The snapshot push is written
twice, once in each arm of a health test whose two arms are identical.

## `Think` — the goal/plan/action tick

**Contract** — advances the planner by the elapsed time, then advances movement by the same
amount. Returns early after the planner if the stalker died during it.

That is the entire function. Everything a stalker decides is inside the planner, whose
operators and evaluators are chapter 24's other files; and everything it does about those
decisions reaches the world through the movement manager. The ordering matters: plan first,
then move, so that a plan formed this tick takes effect this tick.

**Notes** — a large disabled recovery block surrounds both calls. It caught a planner
failure, reported the failing action, re-set up the planner and retried. It is not compiled.
A rebuild should decide deliberately whether a failed plan is recoverable; the shipped
engine lets it propagate.

## `Die`

**Contract** — tells movement the stalker died, notifies whoever last hit it (which is how
a killer hears the "enemy down" vocalisation and how the squad's corpse registry learns of
the body), selects the death animation, plays the death or anomaly-death vocalisation if
death sounds are enabled, and decides — with probability one in three, and only when the
weapon is not slung — whether the corpse's weapon is left with its hammer clutched. Then it
makes the inventory slots unusable and **destroys every magazine of the active weapon's
ammunition type** in the corpse's inventory.

**Invariants** — that last step is a loot-economy decision, not a bug: a dead stalker's
weapon is lootable but its spare ammunition is not, which is what stops the player farming
ammunition from firefights. A rebuild that omits it changes the game's economy noticeably.

## `net_Export` / `net_Import`

**Contract** — export writes health, a timestamp, a flag byte, position, four angles, the
team/squad/group bytes, the game vertex twice, two distances to that vertex, and the
starting dialogue identifier. Import reads a float and a money value *first*, which export
does not write — the two sides disagree by eight bytes. Unreachable in the shipped build,
which has no working transport; see
[Seam: Networking transport](../../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport).
A rebuild implementing multiplayer must redesign this pair.

## `update_object_handler` / `mt_object_handler_update_allowed`

**Contract** — the weapon-handling planner tick, and the predicate deciding whether it may
run on a worker. The predicate demands that the client state has been updated this frame and
that the world has actually been rendered (or the main menu is up), because the planner
reads bone positions that only exist after rendering.

## `Radius`

**Contract** — the stalker's own radius plus its active weapon's, so that a stalker holding
a rifle occupies more touch space than one holding a pistol. That is what makes a stalker
with a long weapon notice items slightly further away.

## `shedule_Scale`

**Contract** — normally the base class's distance-based update-rate scale; **zero** when the
stalker is flagged as a sniper. Zero is the highest priority the scheduler recognises, so a
sniper updates at full rate no matter how far away it is. Scripts set the flag on stalkers
placed to shoot the player from across a map, which would otherwise be updated too rarely to
aim.

## `aim_target` / `aim_bone_id`

**Contract** — resolves a named bone on a target object and returns its world position. A
stalker can be told, from script, to aim at a specific bone rather than at the target's
centre — used for scripted executions and for aiming at vehicles.

**Notes** — the failure message formats the bone *index* where it means to print the bone
name, so a mis-named bone reports nonsense. Debug builds only.

## `ResetBoneProtections`

**Contract** — reloads immunities and bone protection, either from caller-supplied section
names or from the model's embedded configuration. Used when a stalker changes armour.

**Notes** — the caller's bone-protection section name is *overwritten* by the model's before
it is used, so passing one has no effect. Only the immunities argument works. A rebuild
should honour both.

## `critically_wounded`, `can_fire_right_now`, `get_current_smart_cover`, `get_current_loophole`

**Contract** — four small predicates the planner's evaluators consult. Firing right now
requires the best weapon to be in hand *and* to have a round chambered. The smart-cover
accessors return nothing unless the current and the target cover agree, so a stalker in
transit between covers is treated as in neither.
