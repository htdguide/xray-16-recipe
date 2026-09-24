# src/xrGame/ai/monsters — the creature layer

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../README.md) first: it names the state contract, the selector
pattern and the claim that most creatures are data. This page is about the machinery those
creatures are made of.

Everything non-human in the game is assembled here. The directory holds four separable
things, and a rebuilder should plan to build them in this order.

1. **The state machinery** — the tree contract, the identifier space, and the root.
2. **The control bus** — how a decision becomes movement, animation, heading and a route.
3. **The knowledge layer** — four memories, the managers that reduce them to one answer
   each, and the place objects (home, cover, anomalies).
4. **The abilities** — the optional powers a creature is granted, each built on the same
   three-phase animation shape.

The concrete creatures are in the subdirectories, and the shared behaviours they draw on are
in [`states/`](states/README.md) and [`group_states/`](group_states/README.md).

## 1. The state machinery

[`state.h`](state.h.md) is the contract: one node that can be a leaf, a container, or both.
[`state_inline.h`](state_inline.h.md) is the container half — the tick loop that runs one
child and re-selects when it finishes. [`state_manager.h`](state_manager.h.md) is the
interface the creature's *body* sees its brain through, and it is deliberately the only part
of the machinery that is not a template: the body must be able to hold a brain without
knowing which creature's brain it is.
[`monster_state_manager.h`](monster_state_manager.h.md) is the root — a state that is also
the manager the scheduler drives.

[`state_defs.h`](state_defs.h.md) is the identifier space, and it is more load-bearing than
it looks. Identifiers are **bit-encoded**: each global state owns one bit and its substates
carry that bit plus a small index, so "is this creature attacking" is a mask test rather than
a walk up the tree. The values are reachable from script, which freezes them.

Two guards in the root are worth naming because they are the only things standing between
the brain and a class of crash: a creature is not thought about if it is dead, and not
thought about if the alife simulation no longer has a record of it.

## 2. The control bus

This is the largest and least obvious part of the directory, and the piece a rebuilder is
most likely to under-design.

A behaviour state never moves a creature. It **requests** an abstract action and writes
target parameters. Four *channels* — animation, movement, direction, path — then resolve
those requests, and a per-creature **control manager**
([`control_manager.cpp`](control_manager.cpp.md)) arbitrates which component owns each
channel at a time. That arbitration is the whole point: an ability that seizes the body for a
leap must be able to take the animation and movement channels away from the ordinary drivers
and give them back, without either side knowing about the other.

Each channel has a *resource* (the thing that actually applies the value each frame) and a
*base driver* (the default owner that translates the brain's request). The four pairs:

- **animation** — turns an abstract action plus a gait into a clip, through four tables built
  at load from the creature's animation set; also owns acceleration and braking.
- **movement** — eases linear speed toward the commanded target; owns the creature's
  authored gait speeds.
- **direction** — eases heading and pitch toward their targets.
- **path** — the creature's movement manager wearing a channel's face: it turns "go there"
  or "get away from there" into a route, re-plans it on a cadence, and reports arrival.

Components talk to each other over a creature-local **event bus**
([`monster_event_manager.cpp`](monster_event_manager.cpp.md)) with a fixed event vocabulary,
rather than by calling each other. The bus's one real design decision is that unsubscribing
is deferred, so a handler may remove itself while it is being called.

## 3. The knowledge layer

**Four memories**, each a decaying record with its own retention: enemies, heard sounds,
corpses, hits. Perception is event-driven — the senses push into these — so nothing polls.

**Three reducers over them.** The enemy manager picks *one* enemy and grades its danger. The
corpse manager picks *one* corpse. The hit memory additionally records *which of four sides*
the damage came from, which is what lets a creature retreat from a shooter it has never seen.

That split — memory records, manager decides — is why a creature's selector can be a flat
cascade of yes/no questions.

**Place.** The home object is an authored patrol path or a single vertex surrounded by three
nested radii, and it is consulted by nearly every shared behaviour. The cover manager turns
the level's precomputed per-vertex cover values into two creature-level questions: find me a
hiding place relative to a threat, and tell me which way I am most exposed. The anomaly
detector makes a creature that has touched an anomaly route around it for the next half
minute, by *raising the navigation cost* of the cells involved rather than by forbidding
them.

**The pack.** [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) collects what members want,
decides what each should do, and hands commands back down; a registry owns every pack on the
level, keyed by the team/squad/group triple each creature carries. Members claim navigation
vertices and corpses from it and must release them on every exit. Two prepared arrangements
ship: a fan of distinct approach bearings around a shared enemy, and a scatter around the
leader when nothing is happening.

**Morale** is one number that drifts at a rate chosen by an externally set mode, and answers
one question the brains actually ask. **Motion statistics** detect that a creature has been
commanded to move and is not getting anywhere — jammed on geometry — which is the only
self-diagnosis in the chapter.

## 4. The abilities

Almost every creature power is built on the same shape: **prepare, execute, finalize**, three
clips driven by animation-completion events rather than by a timer
([`anim_triple.cpp`](anim_triple.cpp.md)). A rebuilder who implements that one component
correctly gets the jump, the melee jump, the rotation jump, the run-through attack, the
threat display and the critical-wound collapse nearly for free.

Always-on abilities instead use a **rechargeable budget**
([`energy_holder.cpp`](energy_holder.cpp.md)): it drains while the ability is on and refills
while it is off. Invisibility is that budget plus a visibility ramp; the psy aura is that
budget plus a membership set.

Three abilities are worth singling out because they act on the *player* rather than on the
creature: the anti-aim lurch (punishing a player for holding a weapon steady), the actor hold
(taking the camera away), and the auras (a proximity field driving a screen effect, a looping
sound and a periodic hit). The scanning ability is the inverse — it detects the player by
*movement* rather than by sight or sound.

Telekinesis is the outlier: it owns a set of levitated objects and drives them on two clocks,
a coarse phase machine on the behaviour's tick and a fine one per object per frame.

## What could not be recovered

- **`psy_aura` is declared, complete, and instantiated by nothing.** It would keep a live set
  of whatever stands inside a creature-carried field.
- **`telekinesis_inline.h` is a superseded, template-based controller** with a different
  interface from the one in use, kept beside it.
- **`scanning_ability` is used by exactly one creature** (the burer) and reads as general
  machinery. Whether it was meant for more is not recoverable.
- The animation channel's four tables are built by *naming convention* from the model rather
  than from an authored list, so a clip named slightly differently silently disappears from a
  creature's repertoire.
- The anomaly detector's half-minute memory and the cost multiplier it applies are bare
  constants.

## Twins

| Twin | Role |
|---|---|
| [`ai_monster_bones.cpp`](ai_monster_bones.cpp.md) | Turns named bones toward a target angle on top of the playing animation, holds them there, and eases them back to neutral — the mechanism behind a creature tracking its prey with its head. |
| [`ai_monster_bones.h`](ai_monster_bones.h.md) | Declares the additive bone-rotation layer that lets a creature aim or flinch a body part independently of whatever animation is playing. |
| [`ai_monster_defs.h`](ai_monster_defs.h.md) | The shared vocabulary of every non-human creature: the animation alphabet, the abstract action set, the transition and attack-timing records, and the small value types their managers exchange. |
| [`ai_monster_effector.cpp`](ai_monster_effector.cpp.md) | The player's-eye consequences of being attacked by a creature: a post-process envelope and a camera shake. |
| [`ai_monster_effector.h`](ai_monster_effector.h.md) | Declares the two screen effects a creature inflicts on the player: a post-process wash with an attack/hold/release envelope, and a decaying camera shake. |
| [`ai_monster_motion_stats.cpp`](ai_monster_motion_stats.cpp.md) | Detects that a creature is commanded to move but is not getting anywhere — jammed on geometry, or on another creature. |
| [`ai_monster_motion_stats.h`](ai_monster_motion_stats.h.md) | Declares the short ring of recent positions a creature uses to notice it is not actually moving. |
| [`ai_monster_shared_data.h`](ai_monster_shared_data.h.md) | The block of tuned numbers every creature reads from its configuration section — the data half of "creatures differ by data, not code". |
| [`ai_monster_squad.cpp`](ai_monster_squad.cpp.md) | The creature pack: collects what its members want, decides what they should each do about it, and arbitrates the resources they would otherwise fight over. |
| [`ai_monster_squad.h`](ai_monster_squad.h.md) | Declares the creature pack: the goals members report up, the commands the pack hands back down, and the shared claims that stop two creatures wanting the same thing. |
| [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md) | Spreads a pack around a shared enemy: every member gets a distinct approach bearing, so a pack surrounds its prey instead of queueing behind it. |
| [`ai_monster_squad_manager.cpp`](ai_monster_squad_manager.cpp.md) | Owns every creature pack, addressed by the team/squad/group triple each creature already carries, and runs one pack's coordination when its leader thinks. |
| [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md) | Declares the registry that owns every creature pack on the level and finds a creature's pack from the identity it already carries. |
| [`ai_monster_squad_manager_inline.h`](ai_monster_squad_manager_inline.h.md) | The lazy global accessor for the one creature-pack registry. |
| [`ai_monster_squad_rest.cpp`](ai_monster_squad_rest.cpp.md) | Arranges an unalarmed pack around its leader: scattered ahead, behind and to the sides while it travels, gathered at its position while it rests. |
| [`ai_monster_utils.cpp`](ai_monster_utils.cpp.md) | Two questions asked constantly by creature code: is this entity actually standing where the navigation mesh thinks it is, and where in the world is a named bone. |
| [`ai_monster_utils.h`](ai_monster_utils.h.md) | The small shared arithmetic of the creature layer: angle tests, rate-limited approach, navigation-position validation, and the two configuration-line shapes creatures use. |
| [`anim_triple.cpp`](anim_triple.cpp.md) | Plays a wind-up, a held or repeated body, and a recovery, in that order, driven by animation completion rather than by a clock. |
| [`anim_triple.h`](anim_triple.h.md) | Declares the prepare / execute / finalize animation component — the shape almost every creature ability is built from. |
| [`anomaly_detector.cpp`](anomaly_detector.cpp.md) | Makes a creature that has touched an anomaly path around it for the next half minute, by turning the anomaly into a temporary movement restrictor. |
| [`anomaly_detector.h`](anomaly_detector.h.md) | Declares the component that makes a creature remember the anomalies it has bumped into and route around them for a while. |
| [`anti_aim_ability.cpp`](anti_aim_ability.cpp.md) | A creature that notices it is being aimed at charges up, then lurches the player's camera aside — the mechanic that makes a burer hard to shoot. |
| [`anti_aim_ability.h`](anti_aim_ability.h.md) | Declares the ability that punishes a player for holding a weapon steady on a creature: it builds up a detection level and, when full, throws the player's aim off. |
| [`control_animation.cpp`](control_animation.cpp.md) | Reconciles "the clip this creature should be playing" with "the blend actually running on its skeleton", once a frame, for three body slices — and raises the signals that let a blow land on the right frame. |
| [`control_animation.h`](control_animation.h.md) | Declares the creature animation component: the one place that actually starts clips on a creature's skeleton, and the records the rest of the creature talks to it through. |
| [`control_animation_base.cpp`](control_animation_base.cpp.md) | The creature's animation table and the logic that turns "do this action" into "play this clip": variant selection, conditional substitution, posture transitions, and the attack-timing table that decides which frame of which clip does damage. |
| [`control_animation_base.h`](control_animation_base.h.md) | The default driver of a creature's animation channel: it turns the abstract action the brain asked for into a concrete clip, and keeps the clip's speed, the body's speed and the path in agreement. |
| [`control_animation_base_accel.cpp`](control_animation_base_accel.cpp.md) | Acceleration and braking: how fast a creature may change speed, which clip matches the speed it is actually moving at, and when to start slowing down for the end of the path. |
| [`control_animation_base_load.cpp`](control_animation_base_load.cpp.md) | Builds the four tables that turn a creature's abstract actions into clips: the motion table, the transition list, the action map and the conditional substitutions. |
| [`control_animation_base_update.cpp`](control_animation_base_update.cpp.md) | The per-frame loop that picks the creature's clip from its action and its path, then reconciles the clip's speed, the body's speed and the turn rate. |
| [`control_com_defs.h`](control_com_defs.h.md) | The vocabulary of the monster control bus: which control channels exist, which events travel on it, and the three resources an ability can seize. |
| [`control_combase.h`](control_combase.h.md) | The contract every monster control element satisfies: a lifecycle, an optional resource half, an optional client half, and the four shapes those two halves combine into. |
| [`control_critical_wound.cpp`](control_critical_wound.cpp.md) | The critical-wound collapse: the creature stops dead, plays one clip, and tells itself the state is over when it ends. |
| [`control_critical_wound.h`](control_critical_wound.h.md) | Declares the critical-wound collapse and its payload, implemented in [`control_critical_wound.cpp`](control_critical_wound.cpp.md). |
| [`control_direction.cpp`](control_direction.cpp.md) | The direction resource: it eases the creature's heading and pitch toward their targets each frame, writes the result into the model's transform, and reports when a rotation completes. |
| [`control_direction.h`](control_direction.h.md) | Declares the direction resource and its payload, implemented in [`control_direction.cpp`](control_direction.cpp.md). |
| [`control_direction_base.cpp`](control_direction_base.cpp.md) | The default driver of the direction channel: it holds the heading the creature *wants*, from the path or from a target it is facing, and publishes it into the channel each frame. |
| [`control_direction_base.h`](control_direction_base.h.md) | Declares the base driver of the direction channel, implemented in [`control_direction_base.cpp`](control_direction_base.cpp.md). |
| [`control_jump.cpp`](control_jump.cpp.md) | The jump ability: a four-stage animated leap that seizes the whole body, hands the arc to the physics, lands by detecting its own deceleration, and hits whatever it passes through. |
| [`control_jump.h`](control_jump.h.md) | Declares the jump ability and its payload, implemented in [`control_jump.cpp`](control_jump.cpp.md). |
| [`control_manager.cpp`](control_manager.cpp.md) | The bus itself: it owns one control element per channel, arbitrates capture, routes events, and drives the active set each frame. |
| [`control_manager.h`](control_manager.h.md) | Declares the per-creature control bus implemented in [`control_manager.cpp`](control_manager.cpp.md). |
| [`control_manager_custom.cpp`](control_manager_custom.cpp.md) | The creature's ability roster: it owns whichever abilities its creature was granted, offers each one a staging surface, watches every scheduled tick for the ones that fire on their own, and releases each when it reports done. |
| [`control_manager_custom.h`](control_manager_custom.h.md) | Declares the creature's ability roster, implemented in [`control_manager_custom.cpp`](control_manager_custom.cpp.md). |
| [`control_melee_jump.cpp`](control_melee_jump.cpp.md) | The melee jump: a standing creature with its enemy behind it and within arm's reach spins to face it in one clip. |
| [`control_melee_jump.h`](control_melee_jump.h.md) | Declares the melee-jump ability and its payload, implemented in [`control_melee_jump.cpp`](control_melee_jump.cpp.md). |
| [`control_movement.cpp`](control_movement.cpp.md) | The movement resource: it eases the creature's linear speed toward the commanded target and hands the result to the path builder. |
| [`control_movement.h`](control_movement.h.md) | Declares the movement resource and its payload, implemented in [`control_movement.cpp`](control_movement.cpp.md). |
| [`control_movement_base.cpp`](control_movement_base.cpp.md) | The default driver of the movement channel, and the owner of the creature's authored speed table — the ten named gaits every creature is tuned with. |
| [`control_movement_base.h`](control_movement_base.h.md) | Declares the base driver of the movement channel and the owner of the creature's authored speed table, implemented in [`control_movement_base.cpp`](control_movement_base.cpp.md). |
| [`control_path_builder.cpp`](control_path_builder.cpp.md) | The path resource: it is simultaneously the creature's movement manager and a control channel, so the whole navigation stack of chapter 14 is reachable as one bus resource. |
| [`control_path_builder.h`](control_path_builder.h.md) | Declares the path resource — the creature's movement manager wearing a control-channel face — implemented in [`control_path_builder.cpp`](control_path_builder.cpp.md). |
| [`control_path_builder_base.cpp`](control_path_builder_base.cpp.md) | Lifecycle and event handling for the path-builder base: it subscribes to the navigation stack's three events and turns them into the failure and end-of-path facts the per-frame pass reads. |
| [`control_path_builder_base.h`](control_path_builder_base.h.md) | The default driver of the path channel: it turns "go there" or "get away from there" into a target the navigation stack can actually reach, and decides when a path is stale, finished or hopeless. |
| [`control_path_builder_base_inline.h`](control_path_builder_base_inline.h.md) | The one-line setters of the path-builder base, and the one place the chapter's default path-following parameters are written down. |
| [`control_path_builder_base_path.cpp`](control_path_builder_base_path.cpp.md) | Target resolution: turning a position the state layer asked for into a mesh vertex the creature can actually path to, with a four-stage escalation and a random-wander fallback. |
| [`control_path_builder_base_set.cpp`](control_path_builder_base_set.cpp.md) | The target-setting surface: three ways for the state layer to say where a creature should go, and the reset that clears them. |
| [`control_path_builder_base_update.cpp`](control_path_builder_base_update.cpp.md) | The per-frame pass: classify the state of the current path, decide whether the target needs re-resolving, and publish the result into the channel payload. |
| [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md) | The rotation jump: a running creature that finds its enemy behind it skids to a stop through a turn, then accelerates back out toward the enemy. |
| [`control_rotation_jump.h`](control_rotation_jump.h.md) | Declares the rotation-jump ability and its payload, implemented in [`control_rotation_jump.cpp`](control_rotation_jump.cpp.md). |
| [`control_run_attack.cpp`](control_run_attack.cpp.md) | The run-through attack: a creature already running at its enemy plays a strike clip and builds a line that carries it exactly as far as the clip lasts. |
| [`control_run_attack.h`](control_run_attack.h.md) | Declares the run-through attack, implemented in [`control_run_attack.cpp`](control_run_attack.cpp.md). |
| [`control_sequencer.cpp`](control_sequencer.cpp.md) | Plays a list of clips back to back on the whole body, one per animation-end event, and reports when the list runs out. |
| [`control_sequencer.h`](control_sequencer.h.md) | Declares the clip-list player and its payload, implemented in [`control_sequencer.cpp`](control_sequencer.cpp.md). |
| [`control_threaten.cpp`](control_threaten.cpp.md) | The threat display: the creature stops, faces its enemy, plays a warning clip, and fires a single authored callback partway through it. |
| [`control_threaten.h`](control_threaten.h.md) | Declares the threat-display ability and its payload, implemented in [`control_threaten.cpp`](control_threaten.cpp.md). |
| [`controlled_actor.cpp`](controlled_actor.cpp.md) | Taking the player's camera away: while a creature holds the actor, the camera is eased toward a point the creature chooses and nearly every input command is refused. |
| [`controlled_actor.h`](controlled_actor.h.md) | Declares the actor-hold mixin, implemented in [`controlled_actor.cpp`](controlled_actor.cpp.md). |
| [`controlled_entity.h`](controlled_entity.h.md) | The interface a creature implements to be *takeable*: the contract between a mind-controlling creature and its thralls. |
| [`controlled_entity_inline.h`](controlled_entity_inline.h.md) | The generic implementation of being enthralled: save the allegiance, adopt the controller's, and restore it on every exit path. |
| [`corpse_cover.cpp`](corpse_cover.cpp.md) | The cover rule a creature uses when it wants to drag a corpse somewhere private: among cells in a distance band, prefer the one that is *most* enclosed as seen from where the creature stands. |
| [`corpse_cover.h`](corpse_cover.h.md) | Declares the corpse-hiding cover policy, implemented in [`corpse_cover.cpp`](corpse_cover.cpp.md). |
| [`custom_events.h`](custom_events.h.md) | Two event payloads the creature control components pass to each other: "the multi-part animation changed phase" and "the body bounced off something at this fraction of its speed". |
| [`energy_holder.cpp`](energy_holder.cpp.md) | A rechargeable budget for a creature's always-on ability: it drains while the ability is on, refills while it is off, and can switch itself on and off at two different thresholds. |
| [`energy_holder.h`](energy_holder.h.md) | Declares the rechargeable ability budget, implemented in [`energy_holder.cpp`](energy_holder.cpp.md). |
| [`invisibility.cpp`](invisibility.cpp.md) | An energy budget that drains while a creature is hidden and refills while it is shown, with a burst of flicker covering each transition so the switch is seen rather than instantaneous. |
| [`invisibility.h`](invisibility.h.md) | Declares the invisibility mixin: an energy budget that drains while hidden and recharges while shown, plus the flicker that covers the transition. |
| [`melee_checker.cpp`](melee_checker.cpp.md) | Decides when a creature is close enough to swing, using a head-to-target measurement rather than a centre-to-centre one, and narrowing or widening its own idea of "close enough" based on whether recent swings actually connected. |
| [`melee_checker.h`](melee_checker.h.md) | Declares the melee-range judge: when a creature may start swinging and when it must stop. |
| [`melee_checker_inline.h`](melee_checker_inline.h.md) | The authored numbers of the melee window, the per-bout reset, and the arithmetic that turns the adapting inner edge into a matching outer edge. |
| [`monster_aura.cpp`](monster_aura.cpp.md) | A proximity field around a creature that reaches the player directly: a strength falling off with distance, driving a screen effect, a looping sound whose volume tracks it, and a detector tick whose period shortens as the player closes. |
| [`monster_aura.h`](monster_aura.h.md) | Declares one named aura: a proximity field around a creature that drives a screen effect, a looping sound and a Geiger-like detector tick on the player. |
| [`monster_corpse_manager.cpp`](monster_corpse_manager.cpp.md) | Holds the one corpse a creature is currently interested in: normally the nearest one its memory offers, but overridable by script, in which case memory is ignored until the body is picked clean. |
| [`monster_corpse_manager.h`](monster_corpse_manager.h.md) | Declares the corpse manager: the one corpse a creature is currently interested in, either chosen from memory or forced by script. |
| [`monster_corpse_memory.cpp`](monster_corpse_memory.cpp.md) | The dead bodies a creature has seen recently and still considers worth eating, each with where and when it was seen, kept fresh by an eviction pass and queried as "the nearest one". |
| [`monster_corpse_memory.h`](monster_corpse_memory.h.md) | Declares the corpse memory: the set of dead bodies a creature has seen recently and still considers edible. |
| [`monster_cover_manager.cpp`](monster_cover_manager.cpp.md) | Turns the level's precomputed per-vertex cover values into two creature-level answers: the best place to hide from a threat at a chosen distance, and the most open direction to face. |
| [`monster_cover_manager.h`](monster_cover_manager.h.md) | Declares the creature's two cover queries: find a hiding place relative to a threat, and find the most open direction to face. |
| [`monster_enemy_manager.cpp`](monster_enemy_manager.cpp.md) | The one enemy a creature is engaged with, where it was last located, how dangerous the odds are, and a per-tick read of what that enemy is doing — closing, retreating, standing, or unaware of the creature entirely. |
| [`monster_enemy_manager.h`](monster_enemy_manager.h.md) | Declares the enemy manager: the one enemy a creature is currently engaged with, its last known location, and the read of what that enemy is doing. |
| [`monster_enemy_memory.cpp`](monster_enemy_memory.cpp.md) | The hostiles a creature currently knows about, admitted through four separate channels — sight, proximity, a recent hit, and a dangerous sound — each scored by a danger value that weighs how much it is hated against how near it is. |
| [`monster_enemy_memory.h`](monster_enemy_memory.h.md) | Declares the enemy memory: the hostile entities a creature currently knows about, each scored by a danger value. |
| [`monster_event_manager.cpp`](monster_event_manager.cpp.md) | A creature-local publish/subscribe bus whose one real decision is that unsubscribing is deferred, so a handler may remove itself while the bus is calling it. |
| [`monster_event_manager.h`](monster_event_manager.h.md) | Declares the creature's internal event bus: subscribe, unsubscribe, publish. |
| [`monster_event_manager_defs.h`](monster_event_manager_defs.h.md) | The fixed vocabulary of creature-internal events, and the empty base of whatever payload one carries. |
| [`monster_hit_memory.cpp`](monster_hit_memory.cpp.md) | Who has hurt this creature lately, and from which of four sides, so a creature that is shot from behind can turn the right way without ever having seen the shooter. |
| [`monster_hit_memory.h`](monster_hit_memory.h.md) | Declares the hit memory: who has recently hurt this creature, and from which side. |
| [`monster_home.cpp`](monster_home.cpp.md) | The place a creature belongs to: an authored patrol path or a single vertex surrounded by three nested radii, with six different ways of picking a destination inside it depending on whether the creature is settling, wandering, or backing away from something. |
| [`monster_home.h`](monster_home.h.md) | Declares the home area: the place a creature belongs to, as a set of nested radii around an authored patrol path or a single vertex, and the queries that pick a destination inside it. |
| [`monster_morale.cpp`](monster_morale.cpp.md) | A creature's willingness to fight, as one number that drifts at a rate chosen by an externally-set mode and is nudged by being hit and by landing a hit. |
| [`monster_morale.h`](monster_morale.h.md) | Declares morale: one scalar in the unit interval that rises or falls at a state-dependent rate and decides whether a creature is willing to fight. |
| [`monster_morale_inline.h`](monster_morale_inline.h.md) | The mode setters, the nudge primitive, and the one question the brains actually ask of morale. |
| [`monster_sound_defs.h`](monster_sound_defs.h.md) | The creature vocal vocabulary: which utterances exist, how they preempt one another, and how a species extends the set. |
| [`monster_sound_memory.cpp`](monster_sound_memory.cpp.md) | Turns the engine's sound events into a creature's short-term hearing: a filtered, deduplicated, continuously rescored list of what it has heard, plus a separate one-shot signal that a pack-mate is in trouble. |
| [`monster_sound_memory.h`](monster_sound_memory.h.md) | Declares the heard-sounds memory: the classification of a sound into a danger rank, the score that ranks one sound against another, and the separate "a pack-mate is in trouble" signal. |
| [`monster_state_manager.h`](monster_state_manager.h.md) | The root of every creature's state tree: a state that is also the manager the engine drives, generic over the creature type. |
| [`monster_state_manager_inline.h`](monster_state_manager_inline.h.md) | The bodies of the root state's forwarding methods, and the two guards that decide whether a creature thinks at all this tick. |
| [`monster_velocity_space.h`](monster_velocity_space.h.md) | The named movement gaits, as bit flags that combine into the gait sets a creature's states request. |
| [`psy_aura.cpp`](psy_aura.cpp.md) | Refreshes the field's membership from the creature's current position, but only while the ability's charge allows it. |
| [`psy_aura.h`](psy_aura.h.md) | A creature-carried field that keeps a live set of whatever is standing inside it — declared, complete, and instantiated by nothing in the shipped engine. |
| [`scanning_ability.h`](scanning_ability.h.md) | A creature ability that detects the player by *movement* rather than by sight or sound — it accumulates a score while the player moves nearby, fires once when the score crosses a threshold, and then switches itself off. Declared here, implemented in the companion, and mixed into no creature. |
| [`scanning_ability_inline.h`](scanning_ability_inline.h.md) | The movement-detection accumulator: how the score rises, how it decays, and the process-wide token that stops a roomful of scanners from stacking the same screen effect. |
| [`state.cpp`](state.cpp.md) | The one non-template thing about creature states: the identifier-to-name table the debug overlay reads. |
| [`state.h`](state.h.md) | The contract every creature state satisfies: one node in a tree that can act as a leaf, as a container of other nodes, or as a whole brain, with no type distinction between the three. |
| [`state_defs.h`](state_defs.h.md) | The identifier space every creature state is registered and selected by: a bit-field encoding where the family lives in a high bit and the member in the low bits, so that "is this any kind of attack" is one masked comparison. |
| [`state_inline.h`](state_inline.h.md) | The container half of a state node: the tick loop that runs one child and reselects when it finishes, the entry/exit sequencing that guarantees a displaced child is torn down, and the cascades that reach the whole tree at once. |
| [`state_manager.h`](state_manager.h.md) | The interface a creature's body sees its brain through — the only part of the state machinery that is not a template, and therefore the only part a creature can hold without knowing what kind of brain it has. |
| [`telekinesis.cpp`](telekinesis.cpp.md) | Owns a set of levitated objects and drives them on two different clocks: a coarse phase machine on the behaviour tick, and a force application on every physics step. |
| [`telekinesis.h`](telekinesis.h.md) | Declares the telekinesis controller: the thing that owns a set of levitated objects on behalf of a creature or an anomaly. |
| [`telekinesis_inline.h`](telekinesis_inline.h.md) | A superseded, template-based telekinesis controller with a different interface from the one that ships. Included by nothing; it is dead source that the build description still lists. |
| [`telekinetic_object.cpp`](telekinetic_object.cpp.md) | One levitated object's life: rise against switched-off gravity, hover in a dead band, then be dropped or thrown. |
| [`telekinetic_object.h`](telekinetic_object.h.md) | Declares one levitated object: its phase, its timers, its target height, and the sounds that follow it up and out. |

## Subdirectories

| Directory | Contents |
|---|---|
| [`basemonster/`](basemonster/README.md) | The shared creature base: the bag of sub-objects, the think, the load sequence |
| [`states/`](states/README.md) | The shared behaviour library every solitary creature draws on |
| [`group_states/`](group_states/README.md) | The pack-aware parallel library, used by the dog |
| [`bloodsucker/`](bloodsucker/README.md) | Cloaking, feeding, and the chapter's largest block of unreachable combat |
| [`boar/`](boar/README.md) | The chapter's baseline creature |
| [`burer/`](burer/README.md) | Four arbitrated attacks: gravity, telekinesis, a shield and evasion |
| [`cat/`](cat/README.md) | The boar with a different ladder and a disarmed pounce |
| [`chimera/`](chimera/README.md) | An attack made only of leaps, and two unreachable behaviour trees |
| [`controller/`](controller/README.md) | A fragile ranged creature: psychic attacks and mind control |
| [`dog/`](dog/README.md) | The only creature whose brain is written against its pack |
| [`flesh/`](flesh/README.md) | A near-pure data creature with one inverted rule |
| [`fracture/`](fracture/README.md) | The most data-only creature that still fights |
| [`poltergeist/`](poltergeist/README.md) | An invisible flyer that never touches its target |
| [`pseudodog/`](pseudodog/README.md) | The reference creature, and the psy dog that fights by proxy |
| [`pseudogigant/`](pseudogigant/README.md) | A creature that fights with the ground |
| [`rats/`](rats/README.md) | An older, separate architecture kept alive alongside the rest |
| [`snork/`](snork/README.md) | A leaping ambusher that senses through walls |
| [`tushkano/`](tushkano/README.md) | The simplest creature in the game |
| [`zombie/`](zombie/README.md) | A creature defined by refusing to die |
