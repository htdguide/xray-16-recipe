# src/xrGame/ai/stalker/ai_stalker_script.cpp

> Exports the stalker planner's entire vocabulary to Lua: forty-one world properties, sixty operators and twenty-seven vocalisations, by name.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md) · [`ai_stalker_space.h`](ai_stalker_space.h.md) · [`stalker_decision_space.h`](../../stalker_decision_space.h.md) · [`stalker_planner.h`](../../stalker_planner.h.md) · [Seam: Script binding layer](../../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: one large registration table

## Purpose

This is the most consequential registration file in the game, and it is worth saying why
before describing it.

The shipped Lua scripts do not merely *observe* the stalker's planner — they **extend** it.
A script adds its own operators and its own evaluators into the same planner the engine
built, and it does so by naming the engine's world properties and operator identifiers.
Every quest behaviour, every smart-terrain job, every set-piece in all three games is
written against the names in this file.

So: **these names are frozen by conformance criterion 10, and so are their numeric values.**
A script that sets a precondition on "property_enemy" is comparing against the integer this
file exported. Renumbering the enumeration silently breaks every shipped script. This is
unlike the sound *masks* of [`ai_stalker_space.h`](ai_stalker_space.h.md), whose values are
internal and may be changed freely.

## `script_register`

**Contract** — registers one class into the script virtual machine under a name that is not
a class name at all, and hangs three enumerations off it; then registers the stalker itself
as a game-object subclass with a default constructor. Called once at script-layer start-up.

The odd part is the first registration: the stalker's planner type is exported under the
name of a *namespace of identifiers*, so that scripts write the planner's vocabulary as
qualified constants rather than as globals. The class is exported for its constant table,
not for its methods.

### The world properties (forty-one)

What the planner can believe about the world. Grouped by what they are about:

```text
existence:  alive · dead · already_dead
off-screen: alife · smart_terrain_task
situation:  items · enemy · danger · pure_enemy · anomaly · inside_anomaly · panic
weapons:    item_to_kill · found_item_to_kill · item_can_kill · found_ammo
readiness:  ready_to_kill · ready_to_detour
combat:     see_enemy · in_cover · looked_out · position_holded · enemy_detoured
            use_suddenness · use_crouch_to_look_out
wounds:     critically_wounded · enemy_critically_wounded
danger kind: danger_unknown · danger_in_direction · danger_grenade · danger_by_sound
danger work: cover_actual · cover_reached · looked_around · grenade_exploded
puzzles:    puzzle_solved
script:     script
```

**Invariants** — the four *danger kind* properties are mutually exclusive in practice and
select which of four danger sub-planners runs; the four *danger work* properties are the
shared intermediate states those sub-planners drive through. That structure — a kind
selector plus shared progress properties — is what lets four danger responses share one set
of operators.

### The operators (sixty)

What the planner can do. The shape is a **planner of planners**: seven of the sixty are
themselves sub-planners, each a complete goal/plan/action machine with its own operators.

```text
top-level planners:
  death_planner · alife_planner · combat_planner · anomaly_planner · danger_planner
  post_combat_wait · script

off-screen life:
  dead · dying · gather_items · no_alife · smart_terrain_task · solve_zone_puzzle
  reach_task_location · accomplish_task · reach_customer_location
  communicate_with_customer

anomalies:
  get_out_of_anomaly · detect_anomaly

arming:
  get_item_to_kill · find_item_to_kill · make_item_killing · find_ammo

combat:
  aim_enemy · get_ready_to_kill · kill_enemy · retreat_from_enemy
  take_cover · look_out · hold_position · get_distance · detour_enemy
  search_enemy · sudden_attack · kill_enemy_if_not_visible
  kill_if_player_on_the_path
  reach_wounded_enemy · prepare_wounded_enemy · kill_wounded_enemy
  critically_wounded · kill_if_enemy_critically_wounded

danger sub-planners (four, one per danger kind):
  danger_unknown_planner · danger_in_direction_planner
  danger_grenade_planner · danger_by_sound_planner

their operators:
  unknown:      take_cover · look_around · search
  in_direction: take_cover · look_out · hold_position · detour · search
  grenade:      take_cover · wait_for_explosion · take_cover_after_explosion
                look_around · search
```

**Invariants** — the danger-by-sound planner is declared but exports no operators of its
own; it reuses the others'. A rebuilder should not conclude it is empty — its operators are
registered elsewhere; only its *name* is here.

### The vocalisations (twenty-seven)

The sound identifiers scripts may ask a stalker to say. One name per value, matching
[`ai_stalker_space.h`](ai_stalker_space.h.md), plus a dedicated `sound_script` slot for
lines the script layer registers itself.

## Notes

**One binding is wrong and reachable from script.** The name `sound_enemy_killed_or_wounded`
is bound to the interruption **mask** of that vocalisation rather than to its **identifier**.
The mask is a large bit pattern; the identifier is a small index. A script asking a stalker
to say that line passes a number that is not a valid sound identifier. What the sound player
does with it is undefined and a rebuilder must test it against a running original rather
than assume — this is the kind of divergence that would otherwise be discovered only by a
player noticing a missing line.

**`property_dummy` and the walking-in-danger sound are absent.** The enumeration's terminator
is not exported, and the disabled walking-in-danger vocalisation is commented out here as
well as in the header. Both absences are correct.
