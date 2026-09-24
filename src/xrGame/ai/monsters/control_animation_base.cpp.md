# src/xrGame/ai/monsters/control_animation_base.cpp

> The creature's animation table and the logic that turns "do this action" into "play this clip": variant selection, conditional substitution, posture transitions, and the attack-timing table that decides which frame of which clip does damage.

**Needs** — [`control_animation_base.h`](control_animation_base.h.md) · [`control_animation.h`](control_animation.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`anim_triple.h`](anim_triple.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`control_jump.h`](control_jump.h.md) · [Seam: Graphics device](../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — reached through its declarations in [`control_animation_base.h`](control_animation_base.h.md); callers name that, not this file.
**Tier floor** — T1: resolves clip names against the model's motion bank and reads authored motion definitions in place

## Purpose

This is the layer every creature class in the chapter writes its table into — see [`boar.cpp`](boar/boar.cpp.md) for what a table looks like from the creature's side — and the layer every behaviour state asks for a clip. It sits directly above [`control_animation.cpp`](control_animation.cpp.md), which starts the clip; nothing here touches a blend.

The implementation is split across four files sharing one declaration. This one holds the table's *resolution* logic and the attack-timing table; the sibling files hold the acceleration chains, the table construction, and the per-frame selection. The split is by editing convenience, not by concern — a rebuild should feel free to merge them, but should keep this layer separate from the blend layer below it.

## State

```text
RECORD ControlAnimationBase
  anim_storage    : list<AnimItem>, indexed by logical motion   # one slot per motion,
                                                                # empty where unused
  current         : CurrentAnimationInfo    # which logical motion, which variant,
                                            # when started, its speed ramp, its blend
  previous_motion : logical motion
  replacements    : list<Replacement>       # (flag, from motion, to motion)
  transitions     : list<Transition>        # posture and motion transitions
  action_bindings : map<action, logical motion>
  attack_anims    : list<AttackParam>       # per clip: when it hits and how hard
  special_params  : int (bit flags)         # raised by states, read by the creature
  override_motion : logical motion          # forced clip, or "none"
  override_index  : int                     # forced variant, or "any"
  in_attack       : bool
  fx_last_played  : int
```

```text
RECORD AnimItem
  target_name : text          # the clip name prefix; variants are the prefix plus 0,1,2...
  count       : int           # how many variants the model actually has
  spec_id     : int           # -1 means "pick a variant at random"; otherwise pin this one
  posture     : ENUM { stand, sit, lie }
  velocity    : velocity profile
  effects     : four named impact effects, one per direction

RECORD AttackParam
  clip         : clip identity
  time         : real         # fraction of the clip at which the blow lands
  hit_power    : real
  impulse      : real
  impulse_dir  : direction
  field_of_hit : four angles (yaw from/to, pitch from/to), each clamped to a quarter turn
  dist         : real         # maximum range at the moment of the blow
```

**Invariants** — Every logical motion a creature can reach must have a non-empty slot with `count` greater than zero, or selection fails loudly. A replacement's flag is a live reference into the creature, read at selection time, so the substitution follows the creature's condition rather than being decided once.

## `reinit`

**Contract** — Re-arms the component for a life: resets the action to standing idle, clears the special-parameter flags and the override, counts the model's variants for every table slot, initialises the current-animation record to standing idle at variant zero with no speed override, takes animation control, subscribes to the in-clip signal, and loads the attack-timing table from the section named by the creature's `attack_params` key.

**Notes** — Taking animation control in `reinit` rather than waiting to be given it is why a creature always animates even before any behaviour state runs.

## `UpdateAnimCount` — binding the table to the model

**Contract** — For each table slot that has not been counted yet, probes the model for clips named prefix-0, prefix-1, ... until one is missing, and records how many exist. A slot whose prefix yields no clips at all is reported and *removed*, along with every replacement and action binding that referenced it.

```text
FUNCTION UpdateAnimCount()
  FOR EACH slot IN anim_storage
     IF slot is empty        CONTINUE
     IF slot.count > 0       RETURN            # see Notes
     count = 0
     WHILE the model has a clip named slot.prefix + count
        count = count + 1
     IF count > 0
        slot.count = count
     ELSE
        report the missing animation and mark the slot for removal

  FOR EACH marked slot
     drop the slot
     drop every replacement mentioning it, on either side
     drop the first action binding that resolves to it
```

**Notes** — The early return in the loop is not a `continue`: the first slot found already counted aborts the whole pass. Since the pass runs from `reinit` and every slot is counted together, the effect is "count once per creature", which is what was meant. A rebuild that writes it as a per-slot skip gets the same result for creatures loaded normally and a different one for a creature whose table is extended after first use.

Removing a slot rather than failing is what lets one creature class serve models that lack some of its clips — the burer depends on this heavily, see [`burer.cpp`](burer/burer.cpp.md). Only the *first* action binding to the removed motion is dropped, so an action that maps to a missing clip via a second binding is left pointing at nothing.

## `select_animation` — the resolution

**Contract** — Turns the current logical motion into a concrete clip and publishes it to the blend layer. Called when the previous clip ended and whenever animation control starts. Refuses to interrupt an attack clip except on its own ending.

```text
FUNCTION select_animation(triggered_by_animation_end)
  data = the published animation data block ; IF none RETURN
  IF in_attack AND NOT triggered_by_animation_end   RETURN
  in_attack = (current.motion == attack)

  creature.force_final_animation()       # give the creature a last word

  slot = anim_storage[current.motion]    ASSERT present

  IF an override is in force for this motion
     index = override_index              ASSERT within slot.count
  ELSE IF slot.spec_id is set
     index = slot.spec_id                # a pinned variant
  ELSE
     index = random in [0, slot.count)

  clip = resolve(slot.prefix + index)    FATAL if the model lacks it

  data.whole_body.clip   = clip
  data.whole_body.actual = false         # the blend layer will start it
  data.speed             = current.speed_target

  current.name = slot.prefix + index ; current.index = index
  current.started_at = now()
  current.speed_current = 1 ; current.speed_target = "use the clip's own"
```

**Notes** — Three ways a variant is chosen, in falling priority: a behaviour state's explicit override, an authored pin on the slot, and random choice. The random path is what makes idle creatures look unrepetitive without any per-creature state; the pin exists for slots where one particular variant *means* something (the boar reserves one idle for "look around"); the override is how a state plays an exact clip and times itself off its length.

The attack guard is the rule that a melee clip is never cut short. Without it a state change mid-swing would cancel the blow after the player had already been committed to dodging it.

## `CheckTransition` — getting from one posture to another

**Contract** — Given a motion the creature is in and one it wants, searches the transition table for a chain of transition clips connecting them and, if found, queues that chain as a one-shot sequence. Returns whether anything was queued. Refuses outright if the sequencer is unavailable.

```text
FUNCTION CheckTransition(from, to) -> bool
  IF the sequencer will not accept a start   RETURN false
  queued = false
  cursor = from
  LOOP over the transition table
     a row matches when its `from` matches cursor (by motion or by posture)
       AND its `target` matches `to` (by motion or by posture)
     ON a match
        IF NOT queued   begin a sequence
        append the row's transition clip
        queued = true
        IF the row is marked chained
           cursor = the row's transition clip
           restart the scan from the top
        ELSE
           BREAK
  IF queued   commit the sequence
  RETURN queued
```

**Notes** — Rows match on either a specific motion or a whole posture, which is what lets one row say "from any standing motion to lying down" without enumerating the standing motions. The chained flag is what builds multi-step transitions out of single-step rows — standing to asleep is standing to lying, then lying to asleep — and re-scanning from the top after each link is what lets the rows be authored in any order. There is no cycle guard: a table with a chained cycle loops here forever, which is a constraint on the data, not a defence in the code.

The "skip if aggressive" flag rows carry is read at construction and not here; the retired test is visible as a comment.

## `CheckReplacedAnim`

**Contract** — Walks the replacement list and, for the first row whose source matches the current motion and whose live flag is set, substitutes the target motion. First match wins.

**Notes** — The flags are references into the creature — "am I damaged", "am I turning left while running" — so a substitution follows the creature's condition automatically. First-match-wins means a motion with two competing replacements resolves by table order, which is the creature class's registration order.

## `check_hit` — where a blow lands

**Contract** — Handles the in-clip "hit" signal: look up the attack parameters for that clip and that time fraction, play the attack-hit sound, and decide whether the blow connects. Connects only if the enemy is within the row's range *and* inside its field of hit in both yaw and pitch. Reports the attempt and its outcome to the melee tracker whether or not it connected.

```text
FUNCTION check_hit(clip, time_fraction)
  IF no enemy   RETURN
  params = attack_anims row for (clip, time_fraction)
  play the attack-hit sound

  connects = true
  IF |enemy - self| > params.dist                              connects = false
  IF enemy's yaw is outside [own_yaw + foh.from_yaw,
                             own_yaw + foh.to_yaw]             connects = false
  IF enemy's pitch is outside [own_pitch + foh.from_pitch,
                               own_pitch + foh.to_pitch]       connects = false

  IF connects
     deal params.hit_power with params.impulse along params.impulse_dir
  melee_tracker.on_hit_attempt(connects)
```

**Invariants** — The sound plays whether or not the blow lands — a miss is still audible.

**Notes** — This is the mechanism that makes creature melee feel physical rather than scripted: the damage is checked *at the authored frame*, against the enemy's position *at that moment*, inside a cone authored per clip. A player who steps out during the wind-up is missed. The field of hit is stated as offsets from the creature's own facing, in both axes, so a clip that swings low has a different pitch window from one that lunges.

Feeding every attempt, hit or miss, to the melee tracker is how the creature learns that its attacks are not connecting and the behaviour layer can react.

## `AA_reload` — the attack-timing table

**Contract** — Reads the attack-parameters section: each line names a clip and either a parameter tuple or the name of another section holding several tuples. For every tuple, parses it, stores the row, and registers a signal on that clip at the tuple's time fraction. Silently skips lines naming a clip the model does not have.

```text
FUNCTION AA_reload(section)
  IF the section does not exist   RETURN
  clear attack_anims
  FOR EACH (clip_name, value) line IN section
     clip = resolve(clip_name) ; IF invalid  CONTINUE
     IF value is a single token          # it names another section
        FOR EACH line IN that section
           add_row(clip, parse(that line))
     ELSE
        add_row(clip, parse(value))

FUNCTION add_row(clip, params)
  attack_anims.append(params)
  animation_component.add_anim_event(clip, params.time, hit_signal)
```

**Notes** — The one-token indirection is what lets a clip carry *several* blows: a sweep that hits twice is authored as a clip name pointing at a section with two tuples at different time fractions. Distinguishing the two forms by counting tokens is fragile but is the shipped convention and the data relies on it.

Skipping unknown clips rather than failing is the same tolerance that `UpdateAnimCount` shows, and for the same reason: one class, several games' models.

## `parse_anim_params`

**Contract** — Parses one eleven-field tuple in order: time fraction, damage, impulse, impulse direction (three fields), yaw window (two), pitch window (two), range. Normalises the impulse direction and clamps all four field-of-hit angles to just under a quarter turn either side.

**Notes** — The clamp is a correctness requirement, not tidying: the containment test used later is an "is this angle between these two" test on a circle, which stops being meaningful once a window spans half a turn or more. Data that asks for a wider cone silently gets the clamped one.

## `ValidateAnimation` — keeping the clip honest about the motion

**Contract** — Called each frame. Reconciles what the creature's clip implies about movement with what its path follower is actually doing, and corrects whichever is lying.

```text
FUNCTION ValidateAnimation()
  clip_moves  = the current motion has non-zero linear velocity
  path_moving = the path follower is travelling

  IF path_moving AND clip_moves
     steer by the path's direction
       (reversed when the clip is the corpse drag)
  ELSE IF clip_moves AND NOT path_moving
     current.motion = stand_idle ; stop movement
  ELSE IF path_moving AND NOT clip_moves
     stop movement
  ELSE IF the clip is a standing turn AND the creature is not turning
     current.motion = stand_idle
```

**Notes** — This is what prevents the two most visible creature bugs of the era: sliding, where a walking clip plays while the creature is stationary, and skating, where the creature translates under an idle clip. The rule is that the *clip* is authoritative about intent and the *path follower* about fact, and disagreement is resolved by stopping. The corpse-drag exception exists because that clip faces the creature backwards along its path.

## Query surface

**Contract** — A family of small readers over the table, each a lookup with an assertion that the slot exists:

- `get_animation_info` / `get_animation_length` — resolve a (motion, variant) to a clip and its duration. The first returns whether the slot exists; the second asserts it does. This pair is what every timed behaviour state in the chapter builds its deadlines from.
- `get_animation_hit_time` — the absolute time, in seconds from a clip's start, at which its authored blow lands: the attack row's fraction times the clip's duration. Returns half a second if the clip has no attack row.
- `get_animation_variants_count` — how many variants a motion has.
- `AA_GetParams` — the attack row for a named clip, or for a (clip, time fraction) pair. Fails loudly when absent; a clip that delivers damage must be in the table.
- `GetAnimSpeed`, `IsStandCurAnim`, `IsTurningCurAnim`, `GetState` — read the velocity profile and posture recorded on a slot.
- `get_motion_id` — resolve a (motion, variant) to a clip, choosing the variant the same three ways `select_animation` does when none is given.
- `SetCurAnim` — set the current logical motion directly.

**Notes** — `get_animation_hit_time`'s half-second fallback is a magic constant with no discoverable derivation; it exists so a caller asking about a clip with no authored blow gets a plausible number instead of a fault.

## `FX_Play`

**Contract** — Plays the impact effect for a side of the creature, at an intensity clamped to the unit interval, taken from the *current* motion's four-effect set. Rate-limited to one effect per fifty milliseconds regardless of how many hits arrive.

**Notes** — The effects belong to the motion, not to the creature, which is why the creature classes attach the same four-effect set to almost every table row. The rate limit is what keeps a burst of automatic fire from spawning an effect per bullet.

## `VelocityIndex2Action` / `GetActionFromPath`

**Contract** — `VelocityIndex2Action` maps a velocity profile to the abstract action that should play at it, falling through to the creature's own mapping for profiles it does not know — see `CustomVelocityIndex2Action` in [`chimera.cpp`](chimera/chimera.cpp.md). `GetActionFromPath` reads the velocity authored on the path segment the creature is currently travelling and converts it.

```text
FUNCTION GetActionFromPath() -> action
  action = VelocityIndex2Action(velocity of the current path point)
  IF that velocity is the standing profile AND there is a next point
     IF the creature is not turning (within one degree)
        action = VelocityIndex2Action(velocity of the next point)
  RETURN action
```

**Notes** — The look-ahead is what stops a creature from standing still on a path point authored as a stop. A stop point normally means "turn here"; once the turn is done, continuing to stand would stall the creature, so it adopts the *next* segment's action a point early. The one-degree tolerance is what "done turning" means.

Two mappings in the table collapse distinct profiles onto the same action — damaged walk and normal walk both become walking, damaged run and normal run both become running — because the damaged variants are reached through the replacement mechanism, not through the action.

## Debug surface

**Contract** — `GetAnimationName` returns a slot's clip prefix; `GetActionName` indexes a fixed table of action names. The action-name table is positional and must stay in step with the action enumeration; nothing enforces that.
