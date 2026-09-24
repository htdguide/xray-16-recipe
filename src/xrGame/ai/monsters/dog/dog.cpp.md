# src/xrGame/ai/monsters/dog/dog.cpp

> The blind dog: a pack creature whose distinctive machinery is a numbered vocabulary of idle
> "flavour" animations the brain can request by number, plus a jump repertoire gated on rank and
> on the enemy being above it.

**Needs** — [`dog.h`](dog.h.md) · [`dog_state_manager.h`](dog_state_manager.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../control_manager_custom.h`](../control_manager_custom.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [Seam: Graphics device — skeletal animation](../../../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Configuration (ltx)](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`dog.h`](dog.h.md)
**Tier floor** — T2: builds the creature's animation table at load and plays clips through the
renderer's animation interface with a completion callback; the table is data-shaped and the
callback is on the render thread's schedule

## Purpose

Most of what a dog does is in the shared base creature and in the pack states of
[`group_states/`](../group_states/README.md). This file supplies the three things that are the
dog's own: the mapping from abstract actions to named animation clips, a *numbered idle-animation
vocabulary* that the pack states drive by index, and the rules for when a dog may jump.

The numbered vocabulary is the interesting one and it is unusual in this chapter. Every other
creature's states request abstract actions (*run*, *eat*, *look around*) and the animation layer
resolves them. The dog additionally has sixteen named clips — sniffing up, sniffing down,
digging, howling, shaking itself, sitting down, scratching while seated, lying down, getting up —
that the states select **by number**, and which are played directly through the animation
interface rather than through the action layer. That is what makes a pack of dogs look like it is
loitering rather than idling, and it is why the dog's states in
[`group_states/`](../group_states/README.md) are full of bare integers.

## State

```text
RECORD Dog EXTENDS BaseMonster, ControlledEntity

  # --- the numbered-animation machine
  current_anim        : int    # which clip of the vocabulary was last requested
  playing_custom_anim : bool   # a vocabulary clip currently owns the animation channel
  custom_anim_ended   : bool   # a clip finished since the last client update
  anim_pending        : bool   # a clip has been requested and not yet finished
  state_check         : bool   # a new clip request is waiting for the brain to act on it
  saved_state         : int    # the global state to resume once the clip sequence completes

  # --- pack and appetite bookkeeping the group states read
  end_state_eat       : bool
  enemy_position      : vector
  smelling_progress   : int    # counts consecutive sniffing walks; -1 means "not sniffing"
  smelling_count      : int    # how many more sniffing walks this run allows, 0..2

  # --- authored in the creature's configuration section
  anim_factor         : int    # percent chance an idle picks from the rare clips
  corpse_use_timeout  : int    # ms another creature must wait before using a corpse we dropped
  min_life_time       : int    # ms of wakefulness before sleep becomes possible
  drive_out_time      : int    # ms of threatening before the dog escalates
  min_sleep_time      : int    # ms of sleep once asleep
  min_move_dist       : int    # the home region's near/far wander band,
  max_move_dist       : int    #   handed to the home component at reset
```

Invariants worth stating: `playing_custom_anim` is true exactly while the animation channel is
script-captured by this creature, and the four timeouts are all authored **in seconds** and
stored **in milliseconds** — the file multiplies on read. A rebuild that stores seconds will have
every dog's sleep cycle a thousand times too short.

If `max_move_dist` is below `min_move_dist` the pair is discarded and both fall back to their
defaults, rather than being swapped. That is a deliberate refusal to guess what a bad section
meant.

## `Load`

**Contract** — read the creature's tuning from its configuration section, then declare the whole
animation vocabulary: the damaged-variant substitutions, the acceleration chains, the
action-to-clip links, the posture transitions and the per-clip velocity profile. Every number
here comes from data; every *name* here is a prefix into the shipped animation bank and is
therefore frozen against the game data.

**Invariants** — the animation table must be complete before the first update, because the action
layer resolves an unmapped action as an error, not as a no-op.

```text
FUNCTION load(section)
  read tuning (see State); all timeouts are seconds in data, milliseconds in memory

  # a damaged dog substitutes clips rather than selecting different actions
  substitute(when damaged:        run -> run_damaged, walk -> walk_damaged)
  substitute(when turning at run: run -> run_turn_left / run_turn_right)

  # acceleration chains: a walk clip may blend up into a run clip
  chain(walk -> run), chain(walk -> run_turn_left), chain(walk -> run_turn_right)
  chain(walk_damaged -> run_damaged)

  FOR EACH (clip_id, clip_name_prefix, velocity_profile, posture) IN the vocabulary
     register_animation(...)

  # postures are a small graph; moving between them costs a transition clip
  transition(sit -> lie), transition(stand -> sit)
  transition(sit -> stand, skip when aggressive)
  transition(lie -> sit,   skip when aggressive)

  link every abstract action to its clip
```

**Notes** — the *skip when aggressive* flag on the two "get up" transitions is the only piece of
character in the transition graph: a dog that is settling down plays the full animation, and a
dog that has just been startled is allowed to teleport between postures rather than spend a
second standing up. A rebuild that makes all transitions mandatory produces packs that cannot
react.

Three clips are registered twice under different identities — the stealth walk reuses the walk
clip at a slower velocity profile, the glide of a jump reuses the left-jump clip, and sleep reuses
an idle. That is data reuse, not an error; the velocity profile attached to the registration is
what differs.

## `reinit`

**Contract** — reset the numbered-animation machine to idle, install the jump animation data, and
hand the home component the wander band read at load.

**Notes** — the rotation-jump data is registered as four bare clip names and an angle, twice: one
set for a quarter turn and one for a half turn. The names are the digits `"1"` through `"8"`,
which are literal clip names in the shipped model, and the commented-out alternative beside them
shows the descriptive names they replaced. This is frozen against the game data and cannot be
tidied.

The melee jump exists only in *Shadow of Chernobyl* compatibility mode; the other two games give
the dog a general jump ability instead. The choice is made once in the constructor, and it is a
genuine behavioural difference between the games rather than a portability detail.

## `UpdateCL`

**Contract** — the per-frame client update. Beyond the base creature's work it does exactly one
thing: if a vocabulary clip finished since the last frame, run the brain *immediately* rather
than waiting for the creature's next scheduled update.

**Invariants** — the early return when the entity is no longer registered in the off-screen
simulation is a guard against updating a creature that is mid-teardown.

**Notes** — the out-of-band brain run is the load-bearing half of the numbered-animation machine.
A vocabulary clip has no duration the brain knows about; the brain requests one and then *stops
deciding*. Without this, the pack would stand still for one whole scheduler interval after every
sniff. A rebuild that drives the brain purely from the scheduler will produce dogs that pause
visibly between idle animations.

## The numbered-animation vocabulary

Four small functions form one mechanism, and it is clearer stated as one.

**`set_current_animation(n)`** records the request and raises the "a request is waiting" flag; the
brain's next pass sees the flag and switches to the custom state.

**`start_animation`** is where the request becomes real: it refuses if another component already
holds the animation channel, captures the channel for the script layer, and plays the clip named
by the current number with a completion callback.

**`animation_end`** is that callback: it clears the playing flag, raises the finished flag that
the client update watches, and releases the channel.

**`anim_end_reinit`** is the forced version of the same thing — used when the brain changes global
state while a clip is playing and needs the channel back without waiting.

```text
FUNCTION get_current_animation() -> clip_name
  # the vocabulary, frozen against the shipped animation bank
  1  -> sniff upward            9  -> sit idle
  2  -> sniff downward         10  -> scratch while seated
  3  -> sniff in a circle      11  -> look around while seated
  4  -> sniff and dig          12  -> stand up
  5  -> howl                   13  -> lie down
  6  -> growl while standing   14  -> rise from lying
  7  -> shake itself           15  -> tear at a corpse
  8  -> sit down               16  -> bark
  anything else -> sniff forward

FUNCTION random_anim() -> int
  IF roll(anim_factor percent)          # anim_factor is authored per section
    RETURN is_night() ? 5 : 4           # howl at night, dig by day
  RETURN random in 1..3                 # otherwise one of the three sniffs
```

**Notes** — the vocabulary is *ordered*, and the pack states exploit the ordering: the rest state
advances a seated dog through clips 8, 9, 10, 11 by incrementing the number, and reverses a
sleeping dog through 13, 12, 7 to wake it. So the numbers are not opaque tags — their adjacency is
part of the contract, and renumbering them changes behaviour. A rebuild should keep the sequence
and can name the members, but must preserve "the next one" as a meaningful operation.

`anim_factor` is the only authored number in the selection: it is the percentage chance that an
idle is one of the two *rare, loud* animations rather than one of the three quiet sniffs. Splitting
that rare case by time of day — howling only at night — is the one place a dog's behaviour depends
on the world clock.

## `is_night`

**Contract** — true when the game clock's hour is at or before 6 or at or after 21. Reads the
level's game time, which is the world clock, not real time.

**Notes** — the boundaries are authored in code, not in data, and they do not match any other
day/night boundary in the engine (the weather system interpolates continuously). A rebuild that
wants one definition of night must pick which of the two wins.

## `CheckSpecParams`

**Contract** — the hook by which a state requests a one-off animation flourish alongside its
ordinary action. Three flags are honoured: *inspect a corpse* runs the check-corpse clip as a
one-shot sequence; *threaten* and *move while smelling* replace the current clip outright.

**Notes** — the difference between running a clip as a *sequence* and *setting* it is real: the
sequence owns the body until it finishes, while setting it is overwritten by the next tick's
action resolution. Corpse inspection must complete; a threat display must not block movement.

## `check_start_conditions`

**Contract** — extends the base creature's answer about whether an external ability may take over.
For the jump ability specifically the dog adds a pack rule: **only the pack leader may jump**,
unless the jump is the aggressive kind.

```text
FUNCTION check_start_conditions(ability)
  IF ability is jump
    IF an enemy exists AND can_use_aggressive_jump(enemy)
      RETURN base answer                 # an enemy above us overrides rank
    IF we are in a squad AND we are not its leader
      RETURN false
  RETURN base answer
```

**Notes** — this is the clearest pack behaviour in the dog's own code, and it is entirely about
readability to the player: a pack in which every member leaps at once reads as chaos, and one in
which the leader leaps and the rest circle reads as a pack. The exception for an elevated enemy
exists because the alternative — a pack of dogs unable to reach a player standing on a crate —
is worse than the visual cost.

## `can_use_agressive_jump`

**Contract** — true when the enemy's height above the creature exceeds a threshold. The threshold
is 0.8 world units, doubled when the enemy is the player *and the player is currently jumping*.

**Notes** — the doubling is an anti-exploit rule with a visible purpose: without it, a player who
jumps is briefly "above" the dog and triggers the leap, so every player hop would summon a pack
leap. Requiring twice the clearance while the player is airborne means the dog leaps at players
standing on things, not at players hopping.

## `get_attack_rebuild_time`

**Contract** — how often, in milliseconds, the path to the enemy is recomputed while attacking:
100 ms plus 25 ms per world unit of distance to the enemy.

**Notes** — this is the dog's distance-adaptive replacement for the base creature's fixed
interval, and it is a direct cost/quality trade: a dog at arm's length re-plans eight times a
second and one across the clearing twice a second. The player only perceives the difference up
close, and the pathfinder's budget is what is being protected. See the search budgets in
[chapter 14](../../../../src/xrAICore/README.md).

## `HitEntityInJump`

**Contract** — deliver the bite that lands mid-leap, with the damage, impulse and impulse
direction taken from the *attack animation's own* parameters rather than from the creature's
section.

**Notes** — attaching hit parameters to a named animation rather than to the creature is what lets
one creature have several attacks of different strength; the animation is the unit of authoring.
The clip name is written here as a literal, so this hit is frozen against the shipped animation
bank.

## `reload`

**Contract** — re-read per-section data after a respawn. Registers the three-part leap animation
set, with a running velocity profile at both ends — except in *Shadow of Chernobyl* mode, where
the dog has no such leap.

## `debug_on_key`

**Contract** — debug-build only: three keys play named clips directly. Carries no behaviour and
need not be rebuilt. The first key's body is commented out and the key logs a message instead,
which is a leftover rather than a decision.
