# src/xrGame/ai/monsters/basemonster/base_monster_startup.cpp

> Everything a creature reads from its configuration section, the sound bank it loads, and the reset that must leave it exactly as a fresh spawn.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`ai_monster_shared_data.h`](../ai_monster_shared_data.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`state_manager.h`](../state_manager.h.md) · [`monster_home.h`](../monster_home.h.md) · [`monster_cover_manager.h`](../monster_cover_manager.h.md) · [`anomaly_detector.h`](../anomaly_detector.h.md) · [`anti_aim_ability.h`](../anti_aim_ability.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`control_movement_base.h`](../control_movement_base.h.md) · [`corpse_cover.h`](../corpse_cover.h.md) · [`cover_evaluators.h`](../../../cover_evaluators.h.md) · [`CharacterPhysicsSupport.h`](../../../CharacterPhysicsSupport.h.md) · [`alife_simulator.h`](../../../alife_simulator.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: configuration reading and object construction; the checksum-keyed sharing is the only layout concern

## Purpose

This file is the concrete answer to the chapter's central claim that **creatures differ by
data**. Everything a creature is tuned by is read here, and a rebuilder holding this page
knows the whole configuration surface of a non-human creature.

It also holds the reinitialisation, which is the least glamorous and most dangerous routine
in the chapter: a creature is reinitialised when it spawns, when it crosses back into the
loaded level from the offline simulation, and when its restrictions change, and anything it
forgets to reset is state leaking across a save/load boundary.

## The load sequence

A creature is brought up in four passes, in this order, and the order is load-bearing.

```text
reload(section)   # sounds, monster type, home; run before Load
Load(section)     # the base's own parameters and sub-object construction
  ... the concrete creature's Load runs here ...
PostLoad(section) # parameters that need the concrete creature's data to exist
reinit()          # every mutable field back to its spawn value
```

`PostLoad` exists because two of its jobs — registering the attack-on-move animations and
the anti-aim animation — need the creature's *velocity table* to be populated, and that
happens in the concrete creature's own load. The base declares that concrete creatures must
call `PostLoad` at the end of their load.

## `Load`

**Contract** — reads the base parameters, constructs the three cover evaluators, loads the
melee checker, morale, physics, settings, control manager, anomaly detector and cover
manager, and conditionally constructs the steering manager. Reads health as an integer.

```text
FUNCTION Load(section)
  remember the section name

  bone_head       = section's "bone_head",      default the human head bone name
  bone_eye_left   = section's "bone_eye_left",  default none
  bone_eye_right  = section's "bone_eye_right", default none

  construct the three cover evaluators over my movement restrictions
  melee_checker.load(section) ; morale.load(section) ; physics.load(section)

  health = section's "Health"                  # read as an integer, then used as a real
  settings_load(section)                       # the shared settings block
  control.load(section)
  anomaly_detector.load(section) ; cover_manager.load()

  rank                  = section's "rank", default 0
  melee_rotation_factor = section's "Melee_Rotation_Factor", default 1.5
  berserk_always        = section's "berserk_always",        default false

  feel_enemy_who_just_hit_max_distance  = section's, default 20
  feel_enemy_max_distance               = section's, default 3
  feel_enemy_who_made_sound_max_distance= section's, default 49

  separate_factor = section's "separate_factor", default 0
  separate_range  = section's "separate_range",  default 0
  IF separate_factor > 0.0001 AND separate_range > 0.01 THEN
    build a steering manager holding one pack-grouping behaviour with
      cohesion = zero, separation = (0, separate_factor, 0), range = separate_range

  load the four auras from the section (radiation with its "inverted" flag set)

  IF the section names a protections section THEN
    skin_armor   = that section's "skin_armor",           default 0
    hit_fraction = that section's "hit_fraction_monster",  default 0.1
  ELSE the skin-armour damage model is off entirely
```

**Invariants** — the steering manager exists only when *both* separation settings are
meaningfully non-zero. A creature with no separation configured has no grouping behaviour
and the per-frame nudge is skipped entirely, which is the common case.

The separation factor is placed in the **vertical** component of the separation vector. The
steering layer's grouping parameters are per-axis, and the nudge zeroes the vertical
component before applying it, so a rebuild must check which axis that layer actually reads.
This looks like a mismatch and nothing in the source resolves it.

**Notes** — the three "feel enemy" distances are the sensing envelope used by the enemy
manager: a creature notices an enemy that just hit it at 20 units, one that made a sound at
49, and one by presence alone at 3. The asymmetry — you are far more detectable when you act
than when you exist — is the whole of creature stealth in this game, and these three numbers
are it.

Health is read as an integer and stored as a real, so creature health is authored in whole
units.

## `PostLoad`

**Contract** — reads the attack-on-move parameters and, when the ability is enabled, registers
the two running-attack animations against the creature's normal run velocity. Then, when the
section names anti-aim effectors, constructs the anti-aim ability, registers it as a control
component, registers its animation against the creature's standing velocity, and loads its
settings.

```text
FUNCTION PostLoad(section)
  attack_on_move.enabled            = section's "aom_enabled",            default false
  attack_on_move.far_radius         = section's "aom_far_radius",         default 9
  attack_on_move.attack_radius      = section's "aom_attack_radius",      default 0.6
  attack_on_move.update_side_period = section's "aom_update_side_period", default 4000
  attack_on_move.prediction_factor  = section's "aom_prediction_factor",  default 1.3
  attack_on_move.prepare_time       = section's "aom_prepare_time",       default 0
  attack_on_move.prepare_radius     = section's "aom_prepare_radius",     default 7
  attack_on_move.max_go_close_time  = section's "aom_max_go_close_time",  default 8

  IF attack_on_move.enabled THEN
    register the left and right run-attack animations
      (names from "aom_animation_left"/"aom_animation_right",
       both defaulting to the same standing run-attack base name)
      against the normal run velocity, in the standing posture

  IF the section names "anti_aim_effectors" THEN
    build the anti-aim ability, register it as a control component,
    register its animation (from "anti_aim_animation") against the standing velocity,
    and load its settings from the same section
```

**Invariants** — an ability exists if and only if its data does. Neither attack-on-move nor
anti-aim has a code path that turns it on; the presence of a configuration key is the switch.
That is the chapter's rule and this routine is where it is enforced.

**Notes** — the attack-on-move radii describe a three-band approach: outside the far radius
the creature closes normally, inside the prepare radius it begins its wind-up, and inside the
attack radius it strikes. The prediction factor is how far ahead of the target's motion the
creature aims. The side-update period of four seconds is how long it commits to attacking
from one side before reconsidering.

Both run-attack animations default to the same name, so a creature that enables the ability
without authoring separate left and right animations gets one animation for both — which
reads as the creature never leaning into its attack.

## `reload`

**Contract** — loads the creature's sound bank, its indoor/outdoor classification and its
home territory, then captures the panic threshold as its default. Runs before `Load`.

**The sound bank** — twelve named sounds, each optional: idle, distant idle, eat,
aggressive, attack hit, take damage, strike, die, die in anomaly, threaten, steal, panic. A
sound absent from the section is simply not loaded, and playing it later does nothing. Each
is registered with a **type** (how the AI perception layer classifies it when other creatures
hear it), a **priority** and a **channel policy**, and is emitted from the creature's head
bone.

**Invariants** — the priority and channel policy together are the sound arbitration. Death
and damage sounds capture every channel at critical and high priority, so a dying creature is
never talked over. The attack-hit sound also captures every channel, one step above the
aggressive sound. The idle sounds sit at the lowest priority on the base channel, with the
distant idle one step above the near one — so when both are eligible the distant one wins,
which is correct because it is only eligible when the player is far away. The strike sound is
channel-independent and can overlap anything.

**The monster type** — universal, indoor or outdoor, read as a string. It is what the alife
simulation consults when deciding where a creature may be placed; nothing in the behaviour
layer reads it.

**Notes** — the home territory is loaded from a section literally named `home`, not from the
creature's own section. So the home *radii* are shared by every creature kind, and only the
home *point* is per-creature. See [`monster_home.h`](../monster_home.h.md).

Capturing the default panic threshold here — after the base classes have read the section
and before anything can override it — is the only reason the scripted override is reversible.

## `reinit`

**Contract** — resets every piece of mutable creature state to its spawn value. Clears all
four memories and both managers, reinitialises the state machine, morale, the control
manager and the anomaly detector, and zeroes about fifteen flags and timestamps. The single
most important property is that it must be **complete**: anything it misses is state that
survives a save/load or an offline round trip.

**Invariants** — the list it resets is, in effect, the authoritative list of what a creature
carries between ticks: the five behaviour flags (damaged, angry, aggressive, asleep,
run-turn), invisibility, forced real speed, script processing, collision-hit suppression,
enemy-transfer suppression, the melee attack state, the berserk timestamp, the previous sound
type, the leader offset and its timestamp, the cached action target vertex, the three
reachability timestamps, and the animation override.

**Notes** — the action target vertex is reset to the invalid marker rather than to zero,
because zero is a legitimate vertex.

## `spawn`

**Contract** — sets the creature's community, runs the inherited spawn, **asserts that the
level has a navigation mesh, a cross table and a compiled game-graph identity**, registers
the creature with its pack, applies per-spawn settings overrides, and if the creature spawned
under script control immediately resets the animation channel and runs the scripted actions.
Then spawns physics and runs one frame and one scheduled pass of the control manager.

**Invariants** — the navigation assertion is the chapter's hard dependency on chapter 14: a
creature cannot exist on a level without a compiled navigation mesh, and the failure is
made loud rather than degrading into creatures that cannot move.

Running the control manager's two passes immediately at spawn is what guarantees a creature
is in a valid animation on the frame it appears, rather than in a default pose for one tick.

## `destroy`

**Contract** — notifies the controlled-entity hook and finalises the state machine **before**
the inherited destruction, then tears down physics and removes the creature from its pack.
The ordering comment in the original is explicit and the reason is the same as at death: a
state holding a control channel must release it while the creature is still whole.

## the settings block

**Contract** — three routines. `settings_read` fills a settings block from a given
configuration file and section; `settings_load` reads the base block from the creature's
section and interns it; `settings_overrides` re-reads on top of it from the spawn record's
own embedded configuration and interns the result.

```text
FUNCTION settings_read(source, section, block)
  read each field of the block by its configuration key
      # when the source is the main configuration, the key is required;
      # when it is a spawn override, a missing key leaves the field alone
  IF this creature has the run-attack ability THEN
    also read the two run-attack distances
  IF the section names an attack effector THEN
    read the whole effector description from that named section

FUNCTION settings_load(section)
  block = settings_read(main configuration, section)
  base_settings = intern(block, keyed by a checksum of its bytes)

FUNCTION settings_overrides()
  block = a copy of base_settings
  IF my spawn record carries a "settings_overrides" section THEN
    settings_read(that record's configuration, "settings_overrides", block)
  current_settings = intern(block, keyed by a checksum of its bytes)
```

**Invariants** — blocks are **interned by the checksum of their raw bytes**, so every
creature loaded from the same section with the same overrides shares one block. That is the
mechanism that keeps a hundred dogs from holding a hundred copies of the same forty numbers.
It requires the block to be a flat record of scalars with no padding surprises — a real
layout dependency, and the one place in this file that is not tier-agnostic. A rebuild that
interns by value rather than by byte checksum gets the same behaviour without the
constraint.

The two-source reading discipline — required keys from the main configuration, optional
keys from the override — is what lets an individual spawned creature differ from its section
in one field without restating the rest.

**Notes** — the overrides pass mutates the *base* block in place before interning the
result. Since the base block is itself shared, a creature with overrides briefly writes
through a block other creatures are using. It is immediately re-interned under a new
checksum, so no creature reads the corrupted state, but it is a genuine hazard and a rebuild
should copy first.

## critical-wound bones

**Contract** — `load_critical_wound_bones` reads up to three animation names from the
creature's section (head, torso, legs), and for each one present, fills the bone-to-wound-type
map from a named section listing bone names. A wound type with no animation contributes no
bones, so a creature that cannot be crippled in the legs simply never has a leg wound.

**Notes** — the bone lists are sections whose *keys* are bone names and whose values are
ignored. That is the configuration format being used as a set.

## `on_before_sell`

**Contract** — when a creature's single inventory item is sold, mark the item's alife record
as savable again. Creature-carried items are marked unsavable while carried, because they are
reconstructed from the creature's spawn record; once traded away they become independent
objects that must persist.
