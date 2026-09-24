# src/xrGame/ai/monsters/controller — the controller

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The controller is a **fragile ranged creature**, and every decision in this directory follows
from that one fact. It cannot survive a straight fight, so its whole tactic is to be
somewhere its enemy is not looking while something else does the damage.

It is also the smallest interesting brain in the chapter — seven global states, no pack, no
home tactics beyond what the shared attack composite supplies — which makes it the best place
to read the selector pattern in isolation.

## What is actually its own

**The psychic hit**, a charged ranged attack with its own screen effector on the victim, run
as an *ability* on the creature rather than as a behaviour state. That placement is why the
attack survives even though the behaviour state written for it was never registered (below).

**Mind control**, which takes over another creature and puts it into the shared *controlled*
state. This is the one creature in the chapter whose behaviour changes other creatures'
behaviour.

**A tube attack** and its own direction and animation layers, because a creature that fights
by staring needs head and torso aiming decoupled from where it is walking.

**One narrow interruption rule.** The controller admits exactly one external ability —
anti-aim evasion — and only while its attack composite is in its *run at the enemy*
substate. Dodging while committed to anything else would either cancel the action or look
like a glitch. Every other creature's default is "ask the active substate"; this flat refusal
is a deliberate narrowing.

**A retreat reachable only from script.** The run-to-cover state is registered under the
*custom* slot, which the automatic selector never picks. Registering a state *is* the
script-visible surface for forcing a creature into it, so a rebuild that prunes states by
reading the selector alone will silently break shipped scripts.

## What could not be recovered

- **The psychic-fire behaviour state is declared, implemented, and included by the attack
  composite — and never instantiated.** The composite registers three substates and this is
  not one of them. Confusingly, the state identifier it was written for *is* used: the pack
  library registers a different, generic state under it. A rebuilder searching by identifier
  will find the wrong thing.
- **The sprint-between-cover state is never registered anywhere.** Its behaviour is a real
  decision — drop the alarmed demeanour while sprinting — but its movement half is a comment
  where the destination logic should be. The shipped retreat folds the same demeanour switch
  into its run-to-cover state, on a coin flip.
- **The retreat's fallback when no cover is found is the first vertex of the level mesh** —
  an arbitrary corner of the map, not "stay put". A controller that cannot find cover walks
  to the origin. The generous search annulus is the only reason this is rarely seen.
- Two constants above the selector — a five-second "enemy hidden" interval and a ten-unit
  distance — are read by nothing. They are the parameters of a find-enemy branch that was
  planned here and lives in the shared attack composite instead.

## Twins

| Twin | Role |
|---|---|
| [`controller.cpp`](controller.cpp.md) | The controller creature: a slow, frail humanoid whose weapons are enthralment, a continuous psi aura, a psi bolt fired from a look, and a set-piece attack that seizes the player's camera. |
| [`controller.h`](controller.h.md) | Declares the controller creature, implemented in [`controller.cpp`](controller.cpp.md). |
| [`controller_animation.cpp`](controller_animation.cpp.md) | Two-part animation for the controller: a torso clip chosen from what it is doing and a legs clip chosen from the angle between where it looks and where it walks. |
| [`controller_animation.h`](controller_animation.h.md) | Declares the controller's two-partition animation driver, implemented in [`controller_animation.cpp`](controller_animation.cpp.md). |
| [`controller_direction.cpp`](controller_direction.cpp.md) | Head and spine aiming: this creature looks at things by rotating two bones, so its gaze and its body can point in different directions. |
| [`controller_direction.h`](controller_direction.h.md) | Declares the controller's head-and-spine aiming driver, implemented in [`controller_direction.cpp`](controller_direction.cpp.md). |
| [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md) | The set-piece psi attack: four clips during which the player's weapons are blocked, the camera is hauled toward the creature, the player is thrown backwards, and psi damage lands — all of it scaled by the player's psi resistance. |
| [`controller_psy_hit.h`](controller_psy_hit.h.md) | Declares the controller's set-piece psi attack, implemented in [`controller_psy_hit.cpp`](controller_psy_hit.cpp.md). |
| [`controller_psy_hit_effector.cpp`](controller_psy_hit_effector.cpp.md) | Dead file: the bodies of the abandoned psi-attack effectors, entirely commented out. |
| [`controller_psy_hit_effector.h`](controller_psy_hit_effector.h.md) | Dead file: an abandoned post-process and camera effector for the psi attack, entirely commented out. |
| [`controller_script.cpp`](controller_script.cpp.md) | Exports the controller creature to the script layer as a bare class name with a default constructor and nothing else. |
| [`controller_state_attack.h`](controller_state_attack.h.md) | Declares the controller's attack state — the composite that owns every sub-state of a fight — implemented in [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md). |
| [`controller_state_attack_camp.h`](controller_state_attack_camp.h.md) | Declares the camping sub-state — the controller waiting in cover, sweeping its gaze between two limits — implemented in [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md). |
| [`controller_state_attack_camp_inline.h`](controller_state_attack_camp_inline.h.md) | The camping sub-state: on entry, find how wide an arc the creature's cover actually affords by tracing against geometry, then sweep the gaze between those limits at random intervals. |
| [`controller_state_attack_fast_move.h`](controller_state_attack_fast_move.h.md) | Declares a sprint-between-cover state for the controller. Nothing registers it. |
| [`controller_state_attack_fast_move_inline.h`](controller_state_attack_fast_move_inline.h.md) | An unfinished sprint state: it runs, and it drops the creature's alarmed demeanour while it does. It never picks a destination, and nothing registers it. |
| [`controller_state_attack_fire.h`](controller_state_attack_fire.h.md) | Declares the controller's stand-and-stare psychic attack. Included by the attack composite, but never instantiated. |
| [`controller_state_attack_fire_inline.h`](controller_state_attack_fire_inline.h.md) | The controller's stand-still psychic-fire state: freeze, stare the enemy down at range, and let the psy attack fire with its cooldown suppressed. |
| [`controller_state_attack_hide.h`](controller_state_attack_hide.h.md) | Declares the controller's run-to-cover state, implemented in [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md). |
| [`controller_state_attack_hide_inline.h`](controller_state_attack_hide_inline.h.md) | The controller breaks contact: pick a covered navigation vertex away from the enemy, run to it, and switch the creature's outward demeanour on the way. |
| [`controller_state_attack_hide_lite.h`](controller_state_attack_hide_lite.h.md) | Declares the "hide only until the enemy loses sight of me" variant, implemented in [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md). |
| [`controller_state_attack_hide_lite_inline.h`](controller_state_attack_hide_lite_inline.h.md) | Run for cover, but stop the moment the enemy can no longer see you — the cheap variant of the controller's retreat. |
| [`controller_state_attack_inline.h`](controller_state_attack_inline.h.md) | The controller's attack brain: defend the home region first, then alternate between closing on the enemy and biting, and stand and turn when the enemy is somewhere the creature cannot reach. |
| [`controller_state_attack_moveout.h`](controller_state_attack_moveout.h.md) | Declares the "steal toward where the enemy was" state, implemented in [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md). |
| [`controller_state_attack_moveout_inline.h`](controller_state_attack_moveout_inline.h.md) | Leave cover when the enemy is out of sight and creep to where it was, in two hops, glancing around on the way. |
| [`controller_state_control_hit.h`](controller_state_control_hit.h.md) | Declares the controller's mind-control strike state, implemented in [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md). |
| [`controller_state_control_hit_inline.h`](controller_state_control_hit_inline.h.md) | The controller's mind-control strike: start a three-part animation, break it at a fixed moment to deliver the hit, then wait for the animation to finish. |
| [`controller_state_manager.cpp`](controller_state_manager.cpp.md) | The controller's top-level brain: seven global states, a strict priority ordering over them, and one rule about when an external ability may interrupt. |
| [`controller_state_manager.h`](controller_state_manager.h.md) | Declares the controller's brain, implemented in [`controller_state_manager.cpp`](controller_state_manager.cpp.md). |
| [`controller_state_panic.h`](controller_state_panic.h.md) | A declared-but-unbuildable controller panic state: three substate names and nothing behind them. |
| [`controller_tube.h`](controller_tube.h.md) | Declares the wrapper state that hands the creature over to the psychic-attack ability, implemented in [`controller_tube_inline.h`](controller_tube_inline.h.md). |
| [`controller_tube_inline.h`](controller_tube_inline.h.md) | Yield the creature to its psychic-attack ability and stand still until the ability says it is finished. |
