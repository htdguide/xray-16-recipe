# src/xrGame/ai/monsters/controller/controller.cpp

> The controller creature: a slow, frail humanoid whose weapons are enthralment, a continuous psi aura, a psi bolt fired from a look, and a set-piece attack that seizes the player's camera.

**Needs** — [`controller.h`](controller.h.md) · [`controller_animation.h`](controller_animation.h.md) · [`controller_direction.h`](controller_direction.h.md) · [`controller_psy_hit.h`](controller_psy_hit.h.md) · [`controller_state_manager.h`](controller_state_manager.h.md) · [`../controlled_entity.h`](../controlled_entity.h.md) · [`../controlled_actor.h`](../controlled_actor.h.md) · [`../control_animation_base.h`](../control_animation_base.h.md) · [`../control_movement_base.h`](../control_movement_base.h.md) · [`../control_path_builder_base.h`](../control_path_builder_base.h.md) · [`../basemonster/base_monster.h`](../basemonster/base_monster.h.md) · [`../ai_monster_effector.h`](../ai_monster_effector.h.md) · [`../monster_velocity_space.h`](../monster_velocity_space.h.md) · [`../../../Actor.h`](../../../Actor.h.md) · [`../../../ActorCondition.h`](../../../ActorCondition.h.md) · [`../../../ActorEffector.h`](../../../ActorEffector.h.md) · [`../../../character_community.h`](../../../character_community.h.md) · [Seam: Audio device](../../../../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Configuration](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a creature class; the per-frame work is a screen overlay and a sound effector

## Purpose

One concrete creature, built on the base monster of chapter 24 and the control bus of this
slice. Almost everything it does mechanically is inherited; what is distinctive is a short
list, and that list is the twin.

1. **It enthrals.** Any entity that implements the takeable interface and is currently its
   enemy becomes its thrall, up to an authored maximum count. See
   [`../controlled_entity.h`](../controlled_entity.h.md) for what that means — an allegiance
   swap, not a brain replacement.
2. **It aims with its head, not its body.** It has its own direction driver that rotates the
   spine and head bones independently of the body, so it can walk one way while staring
   another. Its psi attacks are gated on the *head's* orientation. See
   [`controller_direction.cpp`](controller_direction.cpp.md).
3. **It animates in two halves.** It has its own animation driver that selects a torso clip
   and a legs clip separately, choosing the legs clip from the angle between where it is
   looking and where it is walking. See
   [`controller_animation.cpp`](controller_animation.cpp.md).
4. **It fires a psi bolt by looking.** When its head is aimed within a few degrees of a
   visible enemy and a delay has elapsed, a particle effect is drawn between the two heads
   and a sound plays. The bolt does no damage; its effect on the player is entirely
   perceptual.
5. **It has a set-piece attack.** The "tube": a four-stage animation during which the
   player's weapons are blocked, the camera is dragged toward the creature, the player is
   thrown backwards, and psi damage lands. See
   [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md).
6. **Its presence drains the player.** Any hit it lands on the actor also drains stamina and,
   when the stamina runs out, makes the player drop what they are holding.
7. **It has two mental states** — idle and danger — and switching between them re-selects its
   animation wholesale.

## State

```text
RECORD Controller                        # beyond what the base monster holds
  max_controlled_number    : int         # authored: "Max_Controlled_Count"
  controlled_objects       : list<Entity>
  mental_state             : one of { idle, danger }

  aura_radius, aura_damage : real        # the continuous psi aura
  stamina_hit              : real        # authored, default 0.2
  tube_damage              : real        # authored
  tube_at_once             : bool        # authored; see Notes
  tube_condition_see_duration : int (ms) # authored, default 50
  tube_condition_min_delay    : int (ms) # authored, default 10000
  tube_condition_min_distance : real     # authored, default 10

  psy_fire_start_time  : int (ms)
  psy_fire_delay       : int (ms)        # 2000 by default; settable to zero
  control_effector     : AttackEffector  # the post-process and camera shake of a control hit
  friend_community_overrides : list<text># communities this creature will not attack
  velocity_move_fwd, velocity_move_bkwd : VelocityParam   # its own two gaits
```

**Invariants** — the thrall list is bounded by the authored maximum and every entry is an
entity that implements the takeable interface. Every path that removes a thrall — death,
destruction, the controller's own death, level teardown — goes through
`OnFreedFromControl` or `FreeFromControl`, and both restore the thrall's allegiance. A
thrall that outlives its controller with a swapped allegiance would be permanently hostile
to its own kind, which is the failure this invariant prevents.

## Authored parameters

| Key | Meaning |
|---|---|
| `Max_Controlled_Count` | how many thralls it may hold |
| `sound_control_start`, `sound_control_hit` | the two sounds played *at the victim's head* |
| `control_effector` | names a second section holding the post-process and camera-shake description of a control hit |
| `Control_Hit` | the particle effect drawn between the two heads |
| `Velocity_MoveFwd`, `Velocity_MoveBkwd` | its own two gaits, registered in addition to the ten the base loads |
| `Friend_Community_Overrides` | a comma-separated list of communities it refuses to treat as enemies |
| `tube_damage` | psi damage of the set-piece attack |
| `tube_at_once` | authored and, in the shipped build, dead — see Notes |
| `tube_condition_see_duration`, `tube_condition_min_delay`, `tube_condition_min_distance` | optional, with defaults of 50 ms, 10 s and 10 units |
| `stamina_hit` | optional, default 0.2 |

Plus the named post-process section's own keys: duality, gray, blur, noise intensity and
grain and rate, three colour triples, three timings for the post-process envelope, and four
for the camera shake.

**Notes** — the "at once" variant of the set-piece condition is guarded by a condition that
is constant-false in the shipped source, so the key is read and never acted on. The variant
would fire the attack as soon as the enemy is visible, skipping the sighting-duration and
distance checks. Whether it was disabled for balance or because it was broken is not
recoverable.

## `Load`

**Contract** — read every authored parameter above, create the two sounds played in the
victim's head, create the five fixed-name sounds of the set-piece attack and the aura,
register the creature's clip set, its action mapping and its two posture transitions,
register the wounded substitutions, load the two extra gaits, and parse the friendly-community
list.

**Notes** — the clip registration is the interesting half and it is unusual: this creature
maps the ordinary walk, the run, the damaged walk and the damaged run **all to the same
clip** at the same authored walk speed. The controller does not run. Its threat is
positional, not kinetic, and the identical mapping is how that is expressed — the state
layer may ask for a run and gets a walk.

Two whole alternative clip sets are commented out in the source, one of them a variant in
which the creature moves in its sneaking torso pose throughout. They record abandoned
tuning and carry no decision.

The five set-piece sounds are created from literal asset paths rather than from the
configuration section, so this creature's audio cannot be re-skinned by data alone. Two of
them — the left and right channels of the aura hit — resolve to the *same* asset, which is
either an oversight or a deliberate mono source played to both ears.

## `UpdateControlled`

**Contract** — once per scheduled tick, if the creature has an enemy that implements the
takeable interface, is not already held by anybody, and the thrall list is below its
maximum, take it: set the hold and give it the follow task.

**Notes** — it enthrals its *current enemy*, so a creature it has just been fighting becomes
its bodyguard. There is no range check and no line of sight: the enemy is whatever the base
monster's memory currently nominates.

## `set_controlled_task`

**Contract** — give every thrall the same task. Follow means follow the controller, attack
means attack the controller's current enemy, none clears the object.

## `InitThink`

**Contract** — before the creature's own thinking, merge its thralls' enemy memories into its
own: every thrall that knows about an enemy contributes that enemy, its last known position,
its vertex and when it was last seen.

**Notes** — this is the payoff of the enthralment mechanic and it is easy to miss. A
controller sees through its thralls. It does not need line of sight of its own; it fights
with the sightings of the creatures it has taken, which is what makes it dangerous in the
open and is the reason the thralls are worth the cost.

## `psy_fire` / `can_psy_fire`

**Contract** — the psi bolt. `can_psy_fire` is a *state-changing query*: it requires the
delay to have elapsed, an enemy to exist and be visible right now, and the head's current
heading to be within five degrees of the direction to the enemy — and, on success, it
**stamps the delay timer**. `psy_fire` draws the particle effect between the two heads and
plays the control-hit sound.

**Notes** — the query stamping the timer is a real hazard: whoever asks is committed. The
animation driver asks it once per torso-clip selection and immediately plays the attack
clip, so there is exactly one caller and the coupling holds. A rebuild should split the ask
from the commit.

The bolt's damage is commented out. As shipped, the psi bolt is a particle effect and a
sound; the damage comes from the aura and the set-piece attack. The delay of two seconds and
the five-degree cone are constants in this file.

`set_psy_fire_delay_zero` and `set_psy_fire_delay_default` let a state turn the bolt into a
continuous stream and back.

## `tube_fire` / `can_tube_fire` / `tube_ready`

**Contract** — the set-piece attack. `can_tube_fire` requires an enemy, that the enemy has
been continuously visible for the authored duration, that the attack element's own start
conditions hold, and that the enemy is **at least** the authored minimum distance away.
`tube_fire` activates the attack's channel. `tube_ready` asks the attack element whether its
cooldown has expired.

**Notes** — the distance test is a *minimum*, not a maximum, and it is the tuning that makes
the attack read correctly: the set piece drags the player *toward* the creature, so it must
begin from far enough away for the drag to be visible. A player standing next to the
controller is safe from it.

The sighting-duration requirement of fifty milliseconds by default is barely a requirement
at all; it exists so the attack cannot trigger on a single frame's glimpse.

## `control_hit` / `play_control_sound_start` / `play_control_sound_hit`

**Contract** — the enthralment hit: thirty points of psi damage to the enemy, and if the
enemy is the player, a camera shake and a post-process effector built from the authored
control-effector description, plus the hit sound. The two sound helpers play at the
*victim's* head position, raised one and a half units, rather than at the creature's.

**Notes** — playing the sound at the victim is what makes the effect read as happening inside
the player's head rather than across the room. The damage of thirty is a literal in this
file, unlike every other damage number on this creature.

## `HitEntity`

**Contract** — every hit this creature lands on the actor also drains the authored stamina
amount, and when the actor's remaining stamina falls below that amount, forces the actor to
drop the held item. Then the ordinary hit handling proceeds.

**Notes** — the forced drop tries the inventory's own drop action first and falls back to a
direct drop, which is the difference between a drop the inventory layer animates and one it
does not. God mode exempts the player from both.

## `is_relation_enemy` / `load_friend_community_overrides` / `is_community_friend_overrides`

**Contract** — this creature refuses to treat two classes of entity as enemies: anything whose
configuration section is literally the zombified-stalker one, and any inventory-owning
non-creature whose community appears in its authored override list. Otherwise the base
monster's relation rules apply.

**Notes** — the zombified-stalker exemption is a hard-coded section name, which is the game's
lore encoded as a string comparison: zombified stalkers are the controller's own victims.
The override list generalizes it, and the exclusion of creatures from the override test means
a controller will still fight other monsters of a "friendly" community.

## `UpdateCL`

**Contract** — per frame: advance the sound-shock effector if one is running and delete it
when it finishes; and, while the control screen effect is active, expand one full-screen
image outward and then contract a second one inward over a hundred and fifty milliseconds,
removing both at the end.

**Notes** — the whole screen-effect path is disabled: nothing sets the flag that starts it,
because the two lines that would are commented out at both of their call sites. The code
describes an effect the shipped game does not show. The expansion factor of two and the
hundred and fifty milliseconds are constants here.

## `create_base_controls`

**Contract** — the creature's override of the base monster's control assembly: it supplies its
*own* animation and direction base drivers and takes the standard movement and path drivers.
Those two substitutions are what make this creature different from every other one.

## `TranslateActionToPathParams`

**Contract** — when the brain asks to run or to walk, plan with the walk gait set and prefer
the walk — damaged or not according to the creature's condition — and enable the path. Any
other action falls through to the base monster's mapping.

**Notes** — the same statement as the clip mapping, made again at the path layer: this
creature has no run. A state that asks for one gets a walk planned and a walk animated.

## `head_orientation`

**Contract** — the creature's head orientation, delegated to its own direction driver. The
base monster asks for this when it needs to know where a creature is *looking* as opposed to
where its body points, and the controller is the creature that makes the distinction real.

## `CheckSpecParams`

**Contract** — when the corpse-checking intent bit is set, run the corpse-check clip through
the sequencer.

## `set_mental_state`

**Contract** — switch between idle and danger, and on an actual change, tell the animation
driver to re-select from scratch.

**Notes** — the mental state is set before the base class's own reset during spawn, and the
source says so explicitly: the animation driver reads it during its own reset, so the order
is load-bearing.

## Lifecycle

`reinit` sets the mental state first, resets the actor hold, resets the base monster,
clears the psi-bolt timers, and registers the creature's two extra gaits with the detailed
path manager. `Die` frees every thrall and tells the set-piece attack to abort. `net_Destroy`
frees every thrall. `net_Spawn` does nothing beyond the base.

`FreeFromControl` releases every thrall and empties the list; `OnFreedFromControl` removes
one thrall by swapping the last entry into its slot, so the list has no stable order.

## Debug-only

`show_debug_info` draws a bent line from the creature to each thrall through a raised midpoint,
which is how the enthralment graph is read on screen. `debug_on_key` fires the set-piece
attack from a key and runs a cover-finding experiment from two marked points. `test_covers` is
empty. None of it carries a decision.
