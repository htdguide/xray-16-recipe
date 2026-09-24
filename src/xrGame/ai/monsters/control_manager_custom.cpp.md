# src/xrGame/ai/monsters/control_manager_custom.cpp

> The creature's ability roster: it owns whichever abilities its creature was granted, offers each one a staging surface, watches every scheduled tick for the ones that fire on their own, and releases each when it reports done.

**Needs** — [`control_manager_custom.h`](control_manager_custom.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_jump.h`](control_jump.h.md) · [`control_rotation_jump.h`](control_rotation_jump.h.md) · [`control_melee_jump.h`](control_melee_jump.h.md) · [`control_run_attack.h`](control_run_attack.h.md) · [`control_threaten.h`](control_threaten.h.md) · [`control_critical_wound.h`](control_critical_wound.h.md) · [`control_sequencer.h`](control_sequencer.h.md) · [`anim_triple.h`](anim_triple.h.md) · [`control_animation_base.h`](control_animation_base.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: allocates the ability set at spawn and polls it every scheduled tick

## Purpose

Every creature in this chapter has this element, and what distinguishes one creature from
another is largely *which abilities it adds to it*. It does four things.

It **owns** the abilities: a creature's construction calls `add_ability` once per ability it
wants, and this element allocates the ability object and registers it on its channel.

It **stages** them: each ability gets a small surface here — fill in this data, then capture,
then activate — because the raw sequence is three calls that must happen in order and none
of them is safe alone.

It **fires** the autonomous ones: five of the abilities decide for themselves when they want
to run, and every scheduled tick this element asks each in turn whether its conditions hold
and starts it if they do. That poll is what makes a creature attack without its state layer
saying so.

It **releases** them: it subscribes to every ability's end event and turns each into a
release of that ability's channel. Without that, an ability would finish its clip and hold
the body forever.

## State

```text
RECORD CustomManager
  sequencer, triple_anim              : optional<ControlElement>
  rotation_jump, jump, run_attack     : optional<ControlElement>
  threaten, melee_jump, critical_wound: optional<ControlElement>

  rot_jump_data   : list<RotationJumpPayload>  # a creature may author several variants
  melee_jump_data : MeleeJumpPayload
  threaten_anim   : text
  threaten_time   : real
```

**Invariants** — every field is empty unless the creature asked for that ability, and every
use site tests for presence first. The ability set is fixed at construction and never
changes.

## `add_ability`

**Contract** — allocate the ability named by a channel identifier and register it on that
channel with the control manager. Called once per ability during the creature's
construction. Granting the threat display also clears its clip name and timing, so a
creature that grants it without authoring it is inert rather than broken.

**Notes** — this switch *is* the creature roster. A rebuild should make it a table of
constructors rather than a switch, but the shape — a creature is a base plus a chosen set
of abilities — is the design of the whole chapter.

## `on_start_control` / `on_stop_control` / `on_event`

**Contract** — three parallel switches over the same channel set. Taking a channel subscribes
to that ability's end event; losing it unsubscribes; receiving an end event releases that
channel.

**Notes** — the triple-animation case is the one exception: its event carries a state, and
only the *finished* state releases the channel; the other states are the triad advancing.
Everything else is a plain "I am done".

This three-switch pattern is where a new ability is added, and forgetting one of the three
produces a creature that performs the ability once and then never moves again. A rebuild
should derive all three from one table.

## `update_schedule`

**Contract** — poll the autonomous abilities, in a fixed order: threat display, attack jump,
rotation jump, run-through attack, melee jump. Each poll is skipped if the creature was not
granted that ability.

**Notes** — the order is a priority order and the first ability to fire takes the body, so
everything after it fails its own start check. Threatening outranks attacking, which is what
makes a creature warn before it strikes.

Jumping over physics obstacles is implemented and commented out of the poll; see below.

## `check_attack_jump`

**Contract** — the jump poll. Requires an enemy, no script control of the creature, the
creature's own class-level permission, current line of sight to the enemy, and the jump's
own distance and angle tests. Then stages a jump with a wind-up, aimed at the enemy object,
with in-flight steering enabled.

**Notes** — the line-of-sight requirement is what stops a creature from leaping at an enemy
it merely remembers. Steering is always on for this variant, so a jump started here tracks
the enemy through the arc.

## `check_jump_over_physics`

**Contract** — dead code, preserved because it describes a behaviour the game visibly lacks.
It would walk forward along the creature's path up to six world units, gather physics objects
near each waypoint, and jump over the first one that is large enough, not already underneath
the creature, and within eight degrees of the creature's facing — aiming at the top of the
obstacle's bounding sphere.

**Notes** — it is excluded from the poll, so creatures do not vault crates. The six-unit
look-ahead, the half-unit minimum obstacle radius, the eight-degree cone and the
half-look-ahead exclusion zone are all constants here. Whether it was removed for cost or
because it looked wrong is not recoverable.

## `check_rotation_jump`, `check_run_attack`, `check_melee_jump`, `check_threaten`

**Contract** — the other four polls, each the same three steps: ask the ability whether its
own conditions hold, ask the creature whether it permits the ability right now, then capture
the channel, fill in the payload, and activate.

The rotation jump picks one of its authored variants **uniformly at random** each time, which
is how a creature with several turn-around clips varies them. The threat display copies the
single authored clip and timing. The run-through attack has no payload at all.

**Notes** — the two-part permission — the ability's own conditions and the creature's — is
consistent across all five, and it is the seam a creature's state layer uses to forbid an
ability without removing it. A creature in its "wounded" state, for instance, refuses the
jump through its own check while the jump's conditions still hold.

## The jump staging surface

**Contract** — four spellings, all of which capture the jump channel, fill the payload and
activate:

- `jump(object, template)` — aim at an object, copying every field of a caller-prepared
  template. Refuses while the creature is under script control.
- `jump(template)` — the same from the template's own target. Answers whether it started.
- `jump(position)` — aim at a bare position with the wind-up skipped. Used by scripted and
  forced jumps.
- `script_jump(position, factor)` — aim at a position with an explicit force factor
  overriding the creature's authored jump factor. No script-control refusal, because this
  *is* the script.

All four clear the force factor to its unset value except the last, which is the only way a
caller can set it.

`jump_if_possible` and `check_if_jump_possible` are the guarded forms: they run the creature's
permission, the ability's distance and angle tests, and the channel's start conditions before
staging. `jump_if_possible` additionally lets its caller choose whether to steer toward the
target, whether to listen for the deceleration signal, and whether to test possibility at all.

**Notes** — the script-control refusal on two of the four spellings and not on the other two
is the rule that lets a script puppet a creature without the creature jumping out from under
it. It is stated per call site rather than centrally.

## `load_jump_data`

**Contract** — resolve a creature's four jump clip names on its model and derive the jump's
flag set from which of them are present. This is where a creature's jump *shape* is decided
and it is entirely driven by absence:

```text
FUNCTION load_jump_data(prepare, prepare_in_move, glide, ground, gait_prepare, gait_ground, flags)
  IF the creature has no skeleton THEN RETURN     # it died before loading finished

  flags <- the caller's flags
  prepare clip        <- resolve(prepare)         or invalid if absent
  prepare-in-move clip<- resolve(prepare_in_move) or invalid; if present, SET prepare_in_move
  glide clip          <- resolve(glide)           # required
  ground clip         <- resolve(ground)          or invalid; if absent, SET ground_skip
  IF neither wind-up clip was given THEN SET prepare_skip
  ALWAYS SET glide_play_anim_once
  ALWAYS SET glide_on_prepare_failed
  gait masks <- the caller's
  force factor <- unset
```

**Notes** — the early return on a missing skeleton is documented in the source as fixing a
specific crash: a creature killed during its own load has no model to resolve names against.

Two flags are set unconditionally for every creature: the glide clip never restarts, and a
wind-up that cannot be staged glides anyway rather than declining. So no shipped creature
uses the other settings of either, and the jump's flag vocabulary is wider than its use.

## The triple-animation staging surface

**Contract** — `ta_fill_data` resolves three clip names into a triad record along with
whether it plays once, whether the preparation stage is skipped, and which body resources it
should seize. `ta_activate` checks start conditions, captures, copies the record into the
payload and activates. `ta_is_active` answers whether the triad is running, optionally
whether it is running *this particular* triad — compared by its three clips.
`ta_pointbreak` asks a running triad to leave its middle stage. `ta_deactivate` releases it.

**Notes** — comparing triads by their three clips rather than by identity is how a creature
asks "am I already doing *that* one", which matters because several triads may be staged
from different states.

## The sequencer staging surface

**Contract** — `seq_init` captures the sequencer and empties its clip list; `seq_add` appends
one clip; `seq_switch` activates. `seq_run` is all three for a single clip, guarded by the
start conditions.

**Notes** — the guarded single-clip form is the one every creature actually uses; the
three-step form is for the rare multi-clip sequence. `seq_init` and `seq_add` are *not*
guarded, so a caller that skips the check and builds a list while another ability holds the
body silently writes into nothing — the payload accessor refuses a non-capturer.

## `script_capture` / `script_release`

**Contract** — the script layer's direct hold on a channel: capture it if its start conditions
allow, and release it if this element is the one holding it. The pair is what lets a script
freeze a creature's body without any ability being involved.

## `add_rotation_jump_data` / `fill_rotation_data` / `add_melee_jump_data`

**Contract** — a creature's load calls these to author its turn-around clips. Each rotation
variant takes two left-side clips, two right-side clips, a turn angle and flags; any name may
be omitted and its clip is marked invalid. Variants accumulate into a list and one is drawn
at random per use. The melee jump takes one clip per side and has a single variant.

## `critical_wound`

**Contract** — stage and start the critical-wound collapse with a clip name. Guarded by the
channel's start conditions; a payload that cannot be reached is a silent no-op.

## `remove_links`

**Contract** — forward a destroyed object to the jump so it can drop its target. The jump is
the only ability in this slice that holds a reference to another object across frames.

## `reinit`

**Contract** — reset and clear the rotation-jump variant list, so a respawned creature
re-authors it during its load.
