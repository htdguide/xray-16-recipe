# src/xrGame/ai/monsters/states — the shared state library

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../../README.md) first: it defines the state contract, the
selector cascade, the latch idiom and the parameter-record handoff that every file here
assumes.

This is the largest directory in the chapter and the reason most creatures need almost no
code of their own. **Every behaviour a creature can have that is not unique to it lives
here**, written once against the creature base rather than against any particular animal.
A creature's brain is, in the common case, a choice of which of these to register and in
what order to prefer them.

## How the library is organised

Three layers, and telling them apart is the key to reading the directory.

**Generic leaves** — the small number of things a creature can actually be told to do. Go to
a point. Turn to face a point. Hold one action. Retreat from a point. Face the least covered
direction. Return to a permitted volume. These take a **parameter record**
([`state_data.h`](state_data.h.md)) filled by whichever composite selected them, and they
contain no judgement at all.

**Behaviour composites** — one per global state, each a small tree of leaves with a selector
and a parameter fill. These are the behaviours: *rest*, *eat*, *attack*, *panic*, *hit
response*, *heard a dangerous sound*, *heard an interesting sound*, *answer a call for help*,
*search for a lost enemy*, *serve a smart terrain's job*, *loiter with the pack*, *be
controlled by someone else*, *fall back to the territory*.

**Sub-composites** — the larger behaviours decompose further. The attack behaviour alone has
melee, run-at, run-attack, attack-on-run, camp, steal-out and move-to-home-point beneath it.

## The ideas worth carrying

**Almost every behaviour has a territory branch.** A creature with an authored home region
behaves differently in nearly every state: it flees toward home rather than away from the
threat, it falls back to home under fire, it drifts back to the middle of home when idle,
and it investigates toward home when a sound came from outside it. A creature spawned
without a home takes the other branch everywhere. That single distinction produces two
noticeably different animals from one library, and a rebuilder who implements only the
homeless branch will get creatures that wander off and never come back.

**Behaviours end on conjunctions, not timers.** Fleeing ends when the creature is *both*
fifteen units away *and* fifteen seconds unseen. The search's final leaf never ends at all
and waits for something else to decide. Retreating after a hit ends at fifteen units of
separation. Using distance alone makes a creature that stops in the open; using time alone
makes one that stops mid-stride. The pairing is the design.

**Idle behaviour is a priority list, not a loop.** Peacetime is: obey a smart terrain's job
if you have one, else get inside your permitted volumes, else get back to the middle of your
territory, else do what the pack says, else alternate idling and wandering on a sixty-second
clock. Only the last rung is what a reader would call "idle".

**Cover is claimed, not merely chosen.** Every state that sends a creature to a covered spot
claims that spot from the pack first and releases it on both exits. Idling, retreating under
fire, feeding and falling back all do this, which is what keeps a group from stacking on one
position.

**A corpse is claimed too.** The feeding composite claims the body against the rest of the
pack for the whole sequence — approach, sniff, drag away, eat, retreat, rest — so two
creatures never share a meal.

**Sleep is a perception state, not an animation.** It is entered and left explicitly and
changes what the creature can notice, rather than just what it looks like.

**Smart-terrain service is a two-graph journey.** Walk on the coarse cross-level graph until
you are in the right region, then across the level mesh to the exact cell, then sit there
waiting to be taken in charge. The two halves are separate states because they use different
graphs; see [chapter 14](../../../../xrAICore/README.md).

## What could not be recovered

- **The play-with-a-corpse behaviour is dead.** It is implemented and registered in the
  solitary peacetime composite, and selected by nothing anywhere. An idle creature knocking a
  corpse around for eight seconds is a thing this engine can do and never does.
- **The object-shove leaf is registered by no creature.** It would push a physics object out
  of a creature's way; nothing runs it.
- **The orbit-a-point leaf is dead twice over**: nothing instantiates it, *and* its
  declaration does not include its implementation, whose every statement is commented out.
- **Three states written as test harnesses shipped.** One is the cat's threat display, one is
  the snork's enemy search; the rest of the harness is disabled. A rebuilder must not prune
  them as scaffolding.
- A large number of behaviour constants are fixed in code with no derivation: forty units to
  flee a frightening sound, fifteen units and fifteen seconds to stop panicking, ten units
  past the last known position to charge to, three seconds of looking around, one hundred and
  twenty degrees per search sweep, eight seconds of corpse-play, twenty seconds of feeding,
  thirty units for a drag, sixty seconds per idle cycle, four seconds of snarling. Each is
  reproducible; none is explained.

## Twins

| Twin | Role |
|---|---|
| [`monster_state_attack.h`](monster_state_attack.h.md) | Declares the shared attack behaviour: the nine-way selector every creature that fights hands its combat to. |
| [`monster_state_attack_camp.h`](monster_state_attack_camp.h.md) | Declares the ambush: take a cover point away from the enemy, watch the open ground, and occasionally creep out to where the enemy was last seen. |
| [`monster_state_attack_camp_inline.h`](monster_state_attack_camp_inline.h.md) | The ambush: a creature that can sense the player through walls takes cover well away from them, faces the most exposed direction, and cycles between watching and creeping toward where the player was last seen — until the player is actually visible, or close, or has hurt it. |
| [`monster_state_attack_camp_stealout.h`](monster_state_attack_camp_stealout.h.md) | Declares the ambush's creeping phase: leave cover at a stalk and move to where the enemy was last seen. |
| [`monster_state_attack_camp_stealout_inline.h`](monster_state_attack_camp_stealout_inline.h.md) | Creeping out of an ambush toward the last place the enemy was seen — the half-committed move that turns a passive ambush into a stalk. |
| [`monster_state_attack_inline.h`](monster_state_attack_inline.h.md) | How a creature fights: an ordered chain of eight alternatives, tried top to bottom every tick, each of which is either "continue what I was doing" or "start something new", with melee as the fallback when none of them applies. |
| [`monster_state_attack_melee.h`](monster_state_attack_melee.h.md) | Declares the bite: stand, face the target, play the attack action, and let the creature's melee checker say when to start and when to stop. |
| [`monster_state_attack_melee_inline.h`](monster_state_attack_melee_inline.h.md) | Biting: hold position, turn to face, and turn at two different speeds depending on whether you are already facing the right way. |
| [`monster_state_attack_on_run.h`](monster_state_attack_on_run.h.md) | Declares the circling attack: for creatures that fight while moving, a state that never ends — it orbits the enemy, predicts where the enemy will be, and strikes in passing from alternating sides. |
| [`monster_state_attack_on_run_inline.h`](monster_state_attack_on_run_inline.h.md) | The run-past attack: pick a side, charge *through* the enemy rather than stopping at him, re-pick the side every few seconds, and abandon the pass if it takes more than six. |
| [`monster_state_attack_run.h`](monster_state_attack_run.h.md) | Declares the approach: run at the enemy's navigation vertex, replanning on the creature's own schedule, until close enough to bite. |
| [`monster_state_attack_run_attack.h`](monster_state_attack_run_attack.h.md) | Declares the charge: run through the enemy with the attack marker set on the running animation, so the hit lands as the creature passes. |
| [`monster_state_attack_run_attack_inline.h`](monster_state_attack_run_attack_inline.h.md) | The charge-through: the creature keeps running and sets a marker on its run animation that makes the clip carry a hit. Whether it connects is the animation's business, not the state's. |
| [`monster_state_attack_run_inline.h`](monster_state_attack_run_inline.h.md) | Closing with the enemy: a path request re-issued every tick against the enemy's current navigation vertex, with route extrapolation on so the creature aims where the enemy is going, and a squad command that can override which way it faces when it gets there. |
| [`monster_state_controlled.h`](monster_state_controlled.h.md) | Declares the behaviour of a creature under someone else's control, implemented in [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md). |
| [`monster_state_controlled_attack.h`](monster_state_controlled_attack.h.md) | Declares the enthralled attack, implemented in [`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md). |
| [`monster_state_controlled_attack_inline.h`](monster_state_controlled_attack_inline.h.md) | Attacking under orders: force the enemy manager onto the controller's chosen target every tick, run the ordinary attack behaviour on top of it, and release the forcing on both exits. |
| [`monster_state_controlled_follow.h`](monster_state_controlled_follow.h.md) | Declares the enthralled escort, implemented in [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md). |
| [`monster_state_controlled_follow_inline.h`](monster_state_controlled_follow_inline.h.md) | Following your master: stop once closer than a *randomly drawn* stopping distance, otherwise walk to a random point near him — which is what keeps a group of thralls spread out instead of stacked. |
| [`monster_state_controlled_inline.h`](monster_state_controlled_inline.h.md) | A mind-controlled creature does exactly one of two things, chosen by the task its controller set — and an attack task whose target has died silently reverts to following the controller instead of failing. |
| [`monster_state_eat.h`](monster_state_eat.h.md) | Declares the feeding behaviour and its twenty-second satiety clock, implemented in [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md). |
| [`monster_state_eat_drag.h`](monster_state_eat_drag.h.md) | Declares the solitary creature's corpse-dragging leaf, implemented in [`monster_state_eat_drag_inline.h`](monster_state_eat_drag_inline.h.md). |
| [`monster_state_eat_drag_inline.h`](monster_state_eat_drag_inline.h.md) | Grip a corpse, walk backward dragging it to a covered spot within thirty units, and let go. |
| [`monster_state_eat_eat.h`](monster_state_eat_eat.h.md) | Declares the actual feeding leaf, implemented in [`monster_state_eat_eat_inline.h`](monster_state_eat_eat_inline.h.md). |
| [`monster_state_eat_eat_inline.h`](monster_state_eat_eat_inline.h.md) | Stand at a corpse and convert its food reserve into satiety, one authored slice at a time, for at most twenty seconds. |
| [`monster_state_eat_inline.h`](monster_state_eat_inline.h.md) | The feeding behaviour of a solitary creature: approach a corpse, sniff it, optionally drag it away, eat until no longer hungry, then retreat and rest — with the corpse claimed against the rest of the squad for the whole sequence. |
| [`monster_state_find_enemy.h`](monster_state_find_enemy.h.md) | Declares the lost-contact search composite, implemented in [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md). |
| [`monster_state_find_enemy_angry.h`](monster_state_find_enemy_angry.h.md) | Declares the four-second threat display, implemented in [`monster_state_find_enemy_angry_inline.h`](monster_state_find_enemy_angry_inline.h.md). |
| [`monster_state_find_enemy_angry_inline.h`](monster_state_find_enemy_angry_inline.h.md) | Four seconds of standing still, posturing and snarling, in the middle of a failed search. |
| [`monster_state_find_enemy_inline.h`](monster_state_find_enemy_inline.h.md) | A four-step search after losing sight of an enemy — charge past the last known spot, cast about, snarl, then mill around indefinitely. |
| [`monster_state_find_enemy_look.h`](monster_state_find_enemy_look.h.md) | Declares the casting-about leaf of the lost-contact search, implemented in [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md). |
| [`monster_state_find_enemy_look_inline.h`](monster_state_find_enemy_look_inline.h.md) | A five-step sweep of the area where contact was lost: look, swing 120 degrees to one side, look, swing another 120, look — each swing randomly either a turn in place or a short dash. |
| [`monster_state_find_enemy_run.h`](monster_state_find_enemy_run.h.md) | Declares the charge-to-last-known-position leaf, implemented in [`monster_state_find_enemy_run_inline.h`](monster_state_find_enemy_run_inline.h.md). |
| [`monster_state_find_enemy_run_inline.h`](monster_state_find_enemy_run_inline.h.md) | Run at full speed to a point ten units past where the enemy was last seen, preferring a route through cover. |
| [`monster_state_find_enemy_walk.h`](monster_state_find_enemy_walk.h.md) | Declares the absorbing final leaf of the search, implemented in [`monster_state_find_enemy_walk_inline.h`](monster_state_find_enemy_walk_inline.h.md). |
| [`monster_state_find_enemy_walk_inline.h`](monster_state_find_enemy_walk_inline.h.md) | The search's terminal leaf: stand where you are, keep making aggressive noise, and wait for something else to decide. |
| [`monster_state_hear_danger_sound.h`](monster_state_hear_danger_sound.h.md) | Declares the reaction to a frightening sound, implemented in [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md). |
| [`monster_state_hear_danger_sound_inline.h`](monster_state_hear_danger_sound_inline.h.md) | Heard something frightening: run forty units away from it, turn to face the open ground, and then cower there indefinitely — unless the creature has a home region, in which case go home instead. |
| [`monster_state_hear_int_sound.h`](monster_state_hear_int_sound.h.md) | Declares the reaction to a merely interesting sound, implemented in [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md). |
| [`monster_state_hear_int_sound_inline.h`](monster_state_hear_int_sound_inline.h.md) | Heard something worth a look: walk toward it — or toward home, if it came from outside the territory — then stand and scan the least-covered direction. |
| [`monster_state_help_sound.h`](monster_state_help_sound.h.md) | Declares the answer-a-packmate's-call behaviour, implemented in [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md). |
| [`monster_state_help_sound_inline.h`](monster_state_help_sound_inline.h.md) | A packmate called for help: run to the exact navigation cell the call came from, look around for three seconds, and stop. |
| [`monster_state_hitted.h`](monster_state_hitted.h.md) | Declares the shot-from-nowhere reaction, implemented in [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md). |
| [`monster_state_hitted_hide.h`](monster_state_hitted_hide.h.md) | Declares the break-away leaf, implemented in [`monster_state_hitted_hide_inline.h`](monster_state_hitted_hide_inline.h.md). |
| [`monster_state_hitted_hide_inline.h`](monster_state_hitted_hide_inline.h.md) | Run flat out away from where the shot came from, until fifteen units of ground are between you and it. |
| [`monster_state_hitted_inline.h`](monster_state_hitted_inline.h.md) | Shot by something unseen: break away from where it came from, then creep back toward it, over and over — unless there is a home region to retreat to. |
| [`monster_state_hitted_moveout.h`](monster_state_hitted_moveout.h.md) | Declares the stalk-back leaf, implemented in [`monster_state_hitted_moveout_inline.h`](monster_state_hitted_moveout_inline.h.md). |
| [`monster_state_hitted_moveout_inline.h`](monster_state_hitted_moveout_inline.h.md) | Creep back toward whatever shot you, hopping from one covered spot to the next, walking while far off and stalking once close. |
| [`monster_state_home_point_attack.h`](monster_state_home_point_attack.h.md) | Declares the fall-back-to-territory leaf used during combat and panic, implemented in [`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md). |
| [`monster_state_home_point_attack_inline.h`](monster_state_home_point_attack_inline.h.md) | Fall back into the territory while fighting: claim a covered spot inside it that nobody else in the squad holds, run there, claim the next one on arrival. |
| [`monster_state_home_point_danger.h`](monster_state_home_point_danger.h.md) | Declares the retreat-to-territory composite used by the frightening-sound and shot-from-nowhere behaviours, implemented in [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md). |
| [`monster_state_home_point_danger_inline.h`](monster_state_home_point_danger_inline.h.md) | Frightened and away from home: claim a covered spot inside the territory, run there exactly, turn to face the most exposed direction, and hold — but if the spot was not covered, leave as soon as you arrive. |
| [`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md) | Declares the wander-back-to-the-middle-of-the-territory leaf, implemented in [`monster_state_home_point_rest_inline.h`](monster_state_home_point_rest_inline.h.md). |
| [`monster_state_home_point_rest_inline.h`](monster_state_home_point_rest_inline.h.md) | Drifted out of the middle of your territory while idle — go back, walking if the territory is placid and running if it is aggressive. |
| [`monster_state_move.h`](monster_state_move.h.md) | A one-line base for leaves that move: it guarantees the path builder is prepared before the leaf's first update. |
| [`monster_state_panic.h`](monster_state_panic.h.md) | Declares the flee-from-a-known-enemy behaviour, implemented in [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md). |
| [`monster_state_panic_inline.h`](monster_state_panic_inline.h.md) | Fleeing an enemy it can see: run until fifteen units away and fifteen seconds unseen, pause three seconds facing the open ground, repeat — and break out of the pause instantly if the enemy reappears. |
| [`monster_state_panic_run.h`](monster_state_panic_run.h.md) | Declares the bolt leaf of the panic behaviour, implemented in [`monster_state_panic_run_inline.h`](monster_state_panic_run_inline.h.md). |
| [`monster_state_panic_run_inline.h`](monster_state_panic_run_inline.h.md) | Run flat out away from the enemy until there is both distance and time between you. |
| [`monster_state_rest.h`](monster_state_rest.h.md) | Declares the peacetime behaviour — everything a creature does when nothing is happening — implemented in [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md). |
| [`monster_state_rest_fun.h`](monster_state_rest_fun.h.md) | Declares the play-with-a-corpse leaf, implemented in [`monster_state_rest_fun_inline.h`](monster_state_rest_fun_inline.h.md). **Dead behaviour**: it is registered in the solitary peacetime composite and selected by nothing, anywhere. |
| [`monster_state_rest_fun_inline.h`](monster_state_rest_fun_inline.h.md) | An idle creature knocking a corpse around for eight seconds. **Dead behaviour** — implemented, registered, and selected by nothing. |
| [`monster_state_rest_idle.h`](monster_state_rest_idle.h.md) | Declares the standing-around composite, implemented in [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md). |
| [`monster_state_rest_idle_inline.h`](monster_state_rest_idle_inline.h.md) | Standing around: take a nearby covered spot nobody else in the squad has claimed, walk to it exactly, turn to face the most exposed direction, and rest there. |
| [`monster_state_rest_inline.h`](monster_state_rest_inline.h.md) | Peacetime: obey a smart terrain's job if you have one, else get inside your permitted volumes, else get back to the middle of your territory, else do what the squad says, else alternate idling and wandering on a sixty-second clock. |
| [`monster_state_rest_sleep.h`](monster_state_rest_sleep.h.md) | Declares the sleeping leaf, implemented in [`monster_state_rest_sleep_inline.h`](monster_state_rest_sleep_inline.h.md). |
| [`monster_state_rest_sleep_inline.h`](monster_state_rest_sleep_inline.h.md) | Sleeping: not an animation but a perception state, entered and left explicitly. |
| [`monster_state_rest_walk_graph.h`](monster_state_rest_walk_graph.h.md) | Declares the wander-between-graph-points leaf, implemented in [`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md). |
| [`monster_state_rest_walk_graph_inline.h`](monster_state_rest_walk_graph_inline.h.md) | Wander: let the path system lead you from one game-graph point to the next, walking, indefinitely. |
| [`monster_state_smart_terrain_task.h`](monster_state_smart_terrain_task.h.md) | Declares the go-do-the-job-a-smart-terrain-assigned composite, implemented in [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md). |
| [`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md) | Declares the cross-level walk toward a smart-terrain job, implemented in [`monster_state_smart_terrain_task_graph_walk_inline.h`](monster_state_smart_terrain_task_graph_walk_inline.h.md). |
| [`monster_state_smart_terrain_task_graph_walk_inline.h`](monster_state_smart_terrain_task_graph_walk_inline.h.md) | Walk on the coarse cross-level graph until you are in the same graph cell as the job. |
| [`monster_state_smart_terrain_task_inline.h`](monster_state_smart_terrain_task_inline.h.md) | Serve the job the alife simulation assigned: walk across the game graph to the right region, then across the level to the exact cell, then sit there waiting to be taken in charge. |
| [`monster_state_squad_rest.h`](monster_state_squad_rest.h.md) | Declares the loiter-near-the-leader behaviour, implemented in [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md). |
| [`monster_state_squad_rest_follow.h`](monster_state_squad_rest_follow.h.md) | Declares the follow-the-squad-order behaviour, implemented in [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md). |
| [`monster_state_squad_rest_follow_inline.h`](monster_state_squad_rest_follow_inline.h.md) | Following the pack: walk to the spot the squad order names, and pause there for a couple of seconds whenever you are within a randomly-chosen slack of it. |
| [`monster_state_squad_rest_inline.h`](monster_state_squad_rest_inline.h.md) | Loitering with the pack: flip a coin between standing still for five to ten seconds and ambling to a random spot within twenty units of the leader. |
| [`monster_state_steal.h`](monster_state_steal.h.md) | Declares the stalking-approach leaf, implemented in [`monster_state_steal_inline.h`](monster_state_steal_inline.h.md). |
| [`monster_state_steal_inline.h`](monster_state_steal_inline.h.md) | Creep up on an enemy who has not noticed you, as long as everything stays quiet and you are between four and fifteen units away. |
| [`state_custom_action.h`](state_custom_action.h.md) | Declares the generic parameterized action leaf, implemented in [`state_custom_action_inline.h`](state_custom_action_inline.h.md). |
| [`state_custom_action_inline.h`](state_custom_action_inline.h.md) | The generic action leaf: play what you were told, for as long as you were told, making the noise you were told to make. |
| [`state_custom_action_look.h`](state_custom_action_look.h.md) | Declares the facing variant of the generic action leaf, implemented in [`state_custom_action_look_inline.h`](state_custom_action_look_inline.h.md). |
| [`state_custom_action_look_inline.h`](state_custom_action_look_inline.h.md) | Implements the leaf state "stand still, play this action, face this point, say this" — the simplest thing a creature can be told to do that still has a direction. |
| [`state_data.h`](state_data.h.md) | The parameter records a composite creature behaviour fills in before handing control to one of the generic leaf states. |
| [`state_hide_from_point.h`](state_hide_from_point.h.md) | Declares the retreat leaf state: get away from a position, using cover, until told to stop. |
| [`state_hide_from_point_inline.h`](state_hide_from_point_inline.h.md) | Implements retreat: hand the path builder a "get away from here" request and let it choose the route and the cover. |
| [`state_hit_object.h`](state_hit_object.h.md) | Declares a leaf state that shoves a nearby physics object out of the creature's way — registered by no creature, so it never runs. |
| [`state_hit_object_inline.h`](state_hit_object_inline.h.md) | Implements the unused object-shove: pick one physics object inside a cone in front of the creature and, half a second in, push it away. |
| [`state_look_point.h`](state_look_point.h.md) | Declares the turn-to-face leaf state: rotate toward a point and, by default, finish when the turn is done. |
| [`state_look_point_inline.h`](state_look_point_inline.h.md) | Implements turn-to-face, whose real decision is the two ways it can end: on a clock, or on the turn actually completing. |
| [`state_look_unprotected_area.h`](state_look_unprotected_area.h.md) | Declares the leaf state that makes a cornered creature turn its back to cover and face the open ground it would have to flee across. |
| [`state_look_unprotected_area_inline.h`](state_look_unprotected_area_inline.h.md) | Implements "face where I am most exposed", by asking the level's per-vertex cover data which direction offers the least protection and turning that way. |
| [`state_move_around_point.h`](state_move_around_point.h.md) | Declares a leaf state that would circle a point at a radius — dead twice over: nothing instantiates it, and it does not include its own implementation. |
| [`state_move_around_point_inline.h`](state_move_around_point_inline.h.md) | The orbit state's implementation file: every statement in it is commented out, and no file includes it. |
| [`state_move_to_point.h`](state_move_to_point.h.md) | Declares the two go-to-a-place leaf states: a plain one that paths once, and an extended one that re-paths, prefers cover and can demand an arrival facing. |
| [`state_move_to_point_inline.h`](state_move_to_point_inline.h.md) | Implements both go-to-a-place states; the substance is in when each one decides it has arrived. |
| [`state_move_to_restrictor.h`](state_move_to_restrictor.h.md) | Declares the corrective state that runs when a creature finds itself outside the volume it is allowed to be in. |
| [`state_move_to_restrictor_inline.h`](state_move_to_restrictor_inline.h.md) | Implements the return-to-permitted-space correction: sprint to the nearest accessible navigation cell and stop the moment you are legal again. |
| [`state_test_look_actor.h`](state_test_look_actor.h.md) | Declares three trivial states written for testing; one of them ended up shipping as the cat's threat display. |
| [`state_test_look_actor_inline.h`](state_test_look_actor_inline.h.md) | Implements the three test poses; the one that ships makes the cat stare at the player with a 1.2-second turn hold. |
| [`state_test_state.h`](state_test_state.h.md) | Declares two composite states written as harnesses; the cover one ships as the snork's enemy-search behaviour, the other is disabled everywhere. |
| [`state_test_state_inline.h`](state_test_state_inline.h.md) | Implements the two harness composites; the live one is a two-state loop — walk to your assigned cover cell, then stand on it — and it is the snork's search behaviour. |
