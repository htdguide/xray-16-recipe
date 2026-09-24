# src/xrGame/ai/monsters/control_animation_base.h

> The default driver of a creature's animation channel: it turns the abstract action the brain asked for into a concrete clip, and keeps the clip's speed, the body's speed and the path in agreement.

**Needs** — [`control_combase.h`](control_combase.h.md) · [`ai_monster_defs.h`](ai_monster_defs.h.md) · [`control_animation.h`](control_animation.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md)
**Used by** — [`anti_aim_ability.cpp`](anti_aim_ability.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_anim.cpp`](basemonster/base_monster_anim.cpp.md) · [`base_monster_feel.cpp`](basemonster/base_monster_feel.cpp.md) · [`base_monster_script.cpp`](basemonster/base_monster_script.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`base_monster_think.cpp`](basemonster/base_monster_think.cpp.md) · [`bloodsucker.cpp`](bloodsucker/bloodsucker.cpp.md) · [`boar.cpp`](boar/boar.cpp.md) · [`burer.cpp`](burer/burer.cpp.md) · [`burer_state_attack_gravi_inline.h`](burer/burer_state_attack_gravi_inline.h.md) · [`burer_state_attack_shield_inline.h`](burer/burer_state_attack_shield_inline.h.md) · [`burer_state_attack_tele_inline.h`](burer/burer_state_attack_tele_inline.h.md) · [`cat.cpp`](cat/cat.cpp.md) · _and 26 more_
**Tier floor** — T2: per-frame clip selection and velocity matching for every live creature

## Purpose

This is the base element of the animation channel, and the largest single piece of
machinery a creature inherits. Its implementation is split across three files by topic and
the state lives here, so this twin carries the state record and the shape of the problem;
the algorithms are in
[`control_animation_base_load.cpp`](control_animation_base_load.cpp.md) (building the
tables at load time),
[`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) (the per-frame
selection) and
[`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md) (acceleration,
braking and the speed-matched clip chains). The split is by topic and is a good one; a
rebuild may keep it or merge it freely.

The problem it solves: the brain of a creature does not name animations. It names an
*action* — stand idle, walk forward, run, attack, eat, drag a corpse. The body meanwhile
is being moved by the path builder at whatever speed the path's waypoints call for. This
element must pick a clip for the action, pick a *variant* of that clip appropriate to the
creature's posture and the turn it is making, insert a transition clip if the posture is
changing, and then scale the clip's playback rate so the creature's feet match the ground
speed the physics is actually producing.

## State

```text
RECORD AnimationBase
  action              : Action                  # what the brain asked for, set every tick
  current             : CurrentAnimationInfo    # motion, variant index, blend, speed, start time
  previous_motion     : Motion                  # last motion actually started
  spec_params         : int (bit set)           # posture/intent modifiers; see Notes
  state_attack        : bool                    # true while an attack clip owns the body
  braking_mode        : bool                    # true while decelerating into a path end
  prev_character_velocity : real                # last frame's measured body speed

  anim_storage        : list<AnimationItem>     # indexed by motion identifier
  motions             : map<Action, MotionItem> # action -> motion, plus its turn variants
  transitions         : list<Transition>        # from (motion|posture) -> to, with a bridge clip
  replaced            : list<ReplacedAnim>      # conditional substitutions, see below
  attack_anims        : list<AttackParams>      # per-attack hit window, damage, impulse, field of hit

  override_motion     : optional<Motion>        # a clip forced from script; wins over everything
  override_index      : optional<int>

  accel.chain         : list<list<Motion>>      # ordered speed ranges, see the accel twin
  accel.active        : bool
  accel.enable_braking: bool
  accel.type          : one of { calm, aggressive }
  accel.calm          : real                    # authored: "Accel_Calm"
  accel.aggressive    : real                    # authored: "Accel_Aggressive"

RECORD AnimationItem                            # one motion's authored description
  target_name  : text          # a clip-name prefix; variants are the prefix plus an index
  spec_id      : int           # -1 means "any variant", otherwise selects one family
  count        : int           # how many variants the model actually has
  posture      : one of { stand, sit, lie, upper_stand }
  velocity     : VelocityParam # the ground speed this clip was authored at
  effects      : { front, back, left, right }   # hit-reaction clip names, optional

RECORD VelocityParam                            # authored as one five-value line
  linear         : real
  angular_real   : real       # the turn rate the body may actually use
  angular_path   : real       # the turn rate the path builder plans corners with
  min_factor     : real       # this clip covers speeds in [linear*min, linear*max]
  max_factor     : real

RECORD Transition
  from    : motion or posture   # a flag says which of the two is meant
  target  : motion or posture
  bridge  : Motion              # the clip played between them
  chain   : bool                # may this transition be followed by another
  skip_if_aggressive : bool     # suppressed when the creature is in a hurry

RECORD ReplacedAnim
  flag     : reference to bool  # a live condition on the creature, e.g. "is wounded"
  from, to : Motion
```

**Invariants** — every motion identifier that any action maps to must have an entry in
`anim_storage`; the per-frame path asserts it rather than tolerating a hole, because a
missing clip means the creature freezes rather than misbehaves visibly.

Within one acceleration chain the authored speed ranges must overlap: each clip's
`linear * max_factor` must exceed the next clip's `linear * min_factor`, or a body speed
between them selects nothing. This is checked once at load in a debug build and is the
single most common authoring error in a new creature.

`spec_params` is a bit set the creature refreshes each tick, naming intents that change
clip selection without changing the action: moving backwards, dragging a corpse, checking
a corpse, a rat-swarm attack, being scared, threatening, a back attack, a rotation jump,
a run attack, a psi attack, the upper-body stance, and smelling while walking. Each bit is
consumed by exactly one creature; they are not a general mechanism so much as an
accumulated list of exceptions.

**Notes** — the clip-name convention is load-bearing and frozen by the shipped models: a
motion is authored as a *prefix* and the engine appends a decimal index, so `stand_idle_`
plus three variants means the model contains `stand_idle_0`, `stand_idle_1`,
`stand_idle_2`. The variant count is discovered by probing the model, not declared. The
bare prefix with no index is accepted as a single-variant fallback.

## Exported units

**Table building** (run once, during the creature's load — see
[`control_animation_base_load.cpp`](control_animation_base_load.cpp.md)):

- `AddAnim` — register one motion: its clip prefix, its variant family, the speed it was
  authored at, the posture it belongs to, and optional directional hit-reaction clips. May
  be declared optional, in which case a model lacking the clip declines silently instead
  of failing.
- `AddTransition` — register a bridge clip between two motions, two postures, or one of
  each. Four spellings of one operation.
- `LinkAction` — map an abstract action to a motion, optionally with left-turn and
  right-turn variants and the angle past which they take over.
- `AddReplacedAnim` — register a conditional substitution: while a named condition on the
  creature holds, one motion is played in place of another. This is how "wounded" is
  expressed without doubling every action.

**Per-frame selection** (see
[`control_animation_base_update.cpp`](control_animation_base_update.cpp.md)):
`update_frame`, `SelectAnimation`, `SelectVelocities`, `SetTurnAnimation`,
`CheckVelocityBounce`, `CheckTransition`, `CheckReplacedAnim`, `GetActionFromPath`,
`ValidateAnimation`, `set_animation_speed`.

**Acceleration and braking** (see
[`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md)):
`accel_init`, `accel_load`, `accel_activate`, `accel_deactivate`, `accel_set_braking`,
`accel_get`, `accel_active`, `accel_chain_add`, `accel_chain_get`, `accel_chain_test`,
`accel_check_braking`.

**Attack clips**: `AA_reload` reads the per-attack table from the configuration section,
`AA_GetParams` looks one up by clip name or by (motion, fraction through the clip), and
`check_hit` fires the hit at the authored moment. The attack table is what makes a
creature's damage come from the *animation*, not from a timer: an authored marker inside
the clip says when the claw connects and the table says with how much force and within
what cone.

**Clip queries**: `get_motion_id`, `get_animation_length`, `get_animation_info`,
`get_animation_hit_time`, `get_animation_variants_count`, `GetAnimSpeed`, `IsStandCurAnim`,
`IsTurningCurAnim`, `GetState`.

**Override**: `set_override_animation` (by motion or by clip name), `clear_override_animation`,
`get_override_animation`, `has_override_animation`. A script-facing escape hatch that
freezes clip selection on one clip. Setting an override while one stands is an error;
clearing is always allowed.

**Effects**: `FX_Play` plays a directional hit-reaction clip on a body side with an
amplitude, layered over whatever is playing.

`ScheduledInit` clears the intent bits and disables acceleration — the once-per-spawn
reset that is *not* part of `reinit` because it must run after the creature's own load.

The event data record `SEventVelocityBounce` carries one number: the ratio between the
body's speed this frame and last, negated when the body slowed down. That sign is how a
jump in flight learns it has landed.
