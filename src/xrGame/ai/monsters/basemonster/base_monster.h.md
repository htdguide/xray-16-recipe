# src/xrGame/ai/monsters/basemonster/base_monster.h

> Declares the creature base — the assembly of managers, memories and control channels that every non-human creature is made of, and the virtual questions each creature answers differently.

**Needs** — [`base_monster.cpp`](base_monster.cpp.md) · [`base_monster_inline.h`](base_monster_inline.h.md) · [`CustomMonster.h`](../../../CustomMonster.h.md) · [`ai_monster_defs.h`](../ai_monster_defs.h.md) · [`ai_monster_shared_data.h`](../ai_monster_shared_data.h.md) · [`ai_monster_utils.h`](../ai_monster_utils.h.md) · [`control_manager.h`](../control_manager.h.md) · [`control_manager_custom.h`](../control_manager_custom.h.md) · [`control_sequencer.h`](../control_sequencer.h.md) · [`monster_enemy_memory.h`](../monster_enemy_memory.h.md) · [`monster_corpse_memory.h`](../monster_corpse_memory.h.md) · [`monster_sound_memory.h`](../monster_sound_memory.h.md) · [`monster_hit_memory.h`](../monster_hit_memory.h.md) · [`monster_enemy_manager.h`](../monster_enemy_manager.h.md) · [`monster_corpse_manager.h`](../monster_corpse_manager.h.md) · [`monster_event_manager.h`](../monster_event_manager.h.md) · [`melee_checker.h`](../melee_checker.h.md) · [`monster_morale.h`](../monster_morale.h.md) · [`monster_aura.h`](../monster_aura.h.md) · [`monster_sound_defs.h`](../monster_sound_defs.h.md) · [`step_manager.h`](../../../step_manager.h.md)
**Used by** — [`ActorCondition.cpp`](../../../ActorCondition.cpp.md) · [`Level_bullet_manager_firetrace.cpp`](../../../Level_bullet_manager_firetrace.cpp.md) · [`PHMovementControl.cpp`](../../../PHMovementControl.cpp.md) · [`WeaponBinocularsVision.cpp`](../../../WeaponBinocularsVision.cpp.md) · [`ai_monster_motion_stats.cpp`](../ai_monster_motion_stats.cpp.md) · [`ai_monster_squad.cpp`](../ai_monster_squad.cpp.md) · [`ai_monster_squad_attack.cpp`](../ai_monster_squad_attack.cpp.md) · [`ai_monster_utils.cpp`](../ai_monster_utils.cpp.md) · [`anomaly_detector.cpp`](../anomaly_detector.cpp.md) · [`anti_aim_ability.cpp`](../anti_aim_ability.cpp.md) · [`base_monster.cpp`](base_monster.cpp.md) · [`base_monster_anim.cpp`](base_monster_anim.cpp.md) · [`base_monster_debug.cpp`](base_monster_debug.cpp.md) · [`base_monster_feel.cpp`](base_monster_feel.cpp.md) · _and 68 more_
**Tier floor** — T2: an aggregate of sub-objects with the creature's per-frame and per-think entry points

## Purpose

Declares the surface implemented across the eleven `base_monster_*` files. The declaration
is worth reading on its own for one reason: **its member list is the architecture of the
chapter**. A creature is not a class with behaviour in it; it is a bag of named sub-objects
plus a set of virtual questions, and a concrete creature is built by filling the bag
differently and answering the questions differently.

## What a creature is made of

**Memories** — four, each a decaying record with its own retention period: enemies (20
seconds), heard sounds (20 seconds), known corpses (20 seconds), hits taken (50 seconds).
Retention is set at construction, not from data.

**Managers over those memories** — an *enemy manager* that picks one current enemy out of
the enemy memory and grades its danger, and a *corpse manager* that picks one corpse to go
to. The split matters: memory records what happened, the manager decides what to care about.

**The state manager** — the creature's brain, a pointer filled by the concrete creature with
its own state machine. The base never constructs one.

**Control channels** — four base controllers (animation, movement, path building,
direction) registered with a *control manager* that arbitrates which component owns each
channel at a time, plus a custom-ability manager holding the sequencer, the triple
animation, the critical-wound ability and whatever the creature adds.

**Coordination and place** — the cover manager, the home-territory object, the anomaly
detector, the step (footstep) manager, and the pack the creature is registered in.

**Body and condition** — the character physics support, morale, the melee-reach checker,
four auras (psy, radiation, fire, and a base one) that radiate an influence at nearby
entities, skin armour and its hit fraction, and the critical-wound bone map.

**Settings** — two references to a shared settings block: the base one loaded from the
creature's configuration section, and the current one after per-spawn overrides. Both are
reference-counted and shared between every creature loaded from the same section, keyed by
a checksum of the block's bytes.

**Steering** — an optional grouping behaviour that nudges the creature's position to keep
pack members from standing inside each other. Only created when the creature's section
configures a separation factor and range.

**The anti-aim ability** — optional, only created when the section names effectors for it.

## The virtual questions

Ten *ability* questions each creature answers with a constant: can it turn invisible, drag
corpses, attack with psi, cause an earthquake, jump, feel at a distance, attack on the run,
rotation-jump, jump over physics objects, and whether it needs pitch correction. The default
is no to all but the last. These are not capabilities the base implements; they are
*declarations* other machinery branches on, which is why they are questions and not flags in
the settings block.

Plus: how long before an attack path is rebuilt, how a turn animation is chosen, how the
creature looks at a point, what its class name is for debug variables, whether it can be
seen, whether it should return home when its enemy is unreachable, and whether shots at it
leave marks.

## Exported units

Beyond the above, the declaration exposes the entry points implemented elsewhere:
`Load` / `PostLoad` / `reload` / `reinit` / spawn / destroy, the per-frame and
per-scheduled-update paths, the think, the memory update, the four cover queries, the
hit paths, the script-action assignments, the sound selection, the action-to-path-parameter
translation, and — under a debug build only — a large tree of introspection.

**Notes** — the header carries a block comment reading "Kill From Here", above a run of
public mutable flags (damaged, angry, growling, aggressive, asleep, turning left while
running, turning right while running). They are read and written from everywhere, which is
exactly what the comment is complaining about. A rebuild should make each of them derived
or owned, and the twins say where each is decided.
