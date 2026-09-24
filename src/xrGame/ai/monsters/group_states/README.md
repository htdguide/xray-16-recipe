# src/xrGame/ai/monsters/group_states — the pack-aware state library

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../../README.md) first for the state-tree contract; everything
here satisfies it.

This is a **second, parallel copy of the shared state library** in
[`../states/`](../states/README.md), rewritten so that every decision consults the creature's
pack. Rest, attack, panic, feeding and the dangerous-sound response all have pack versions
here.

Exactly one creature uses it: the [dog](../dog/README.md). No other creature in the game has
a pack-aware brain. That makes this directory unusually valuable, because the difference
between the two libraries is a clean, controlled statement of **what coordination actually
costs and buys** — the same five behaviours, written twice, once for a lone animal and once
for a group.

## What pack-awareness changes

**A shared judgement replaces individual assessment.** The pack carries one bit — *is our
territory threatened* — and the attack state reads that instead of each member deciding for
itself whether to fight. A pack therefore commits or withdraws as a unit, which no collection
of individually reasoning animals would ever do.

**Position is assigned, not chosen.** Where a solitary creature asks the cover system for a
spot, a pack member takes its place from its *index within the pack*: the charge offsets each
member by a wandering vector so they do not converge on one line, the approach fans them
across an arc on the home side, and the idle states claim spots so no two animals stand in
one place.

**Everything happens inside a territory.** The pack states are written in terms of the home
region's three nested radii, not in terms of absolute distances. Fleeing means running
*toward* the territory on the far side from the enemy. Dragging a corpse means hauling it
inward. Panicking means sprinting into the territory and then stopping to stare outward. A
pack does not have a direction to run in; it has a place to be.

**Feeding is governed by a clock the whole pack shares.** Walk to the corpse, drag it into
cover, tear, eat, withdraw and settle, under a twenty-second satiety timer that ends the
whole sequence.

## One piece of machinery worth naming

[`state_adapter.h`](state_adapter.h.md) lets a behaviour be written as a **plain object with
no knowledge of the state tree**, by wrapping it in a node the tree understands. It is the
only place in the chapter where the tree's contract is made optional, and it is how the more
elaborate pack states avoid inheriting machinery they do not use.

## What could not be recovered

- **The pack variant of "fall back to the home region during a fight" is unused.** It is
  complete: when the enemy stands somewhere a creature cannot walk, fall back to a reserved
  cover spot on the enemy's side of the territory and watch the open ground. Nothing selects
  it.
- The twenty-second satiety clock, the three-second panic pause, and the ring radii used by
  the stalking approach are bare constants.
- The mapping from a "flavour" animation *number* to what the clip depicts lives only in the
  game data. The custom state plays them by index and chooses a sound from the same index.

## Twins

| Twin | Role |
|---|---|
| [`group_state_attack.h`](group_state_attack.h.md) | Declares the pack attack brain, implemented in [`group_state_attack_inline.h`](group_state_attack_inline.h.md). |
| [`group_state_attack_inline.h`](group_state_attack_inline.h.md) | The pack attack brain: defend the territory first; if the enemy has earned aggression, charge; otherwise stalk it inward through three shrinking rings and growl at it until it leaves. |
| [`group_state_attack_run.h`](group_state_attack_run.h.md) | Declares the pack charge state, implemented in [`group_state_attack_run_inline.h`](group_state_attack_run_inline.h.md). |
| [`group_state_attack_run_inline.h`](group_state_attack_run_inline.h.md) | The charge: run at where the enemy *will be*, offset by a wandering vector so the pack does not converge on one line, and approach from the direction the squad assigned. |
| [`group_state_custom.h`](group_state_custom.h.md) | Declares the wrapper that plays one numbered "flavour" animation, implemented in [`group_state_custom_inline.h`](group_state_custom_inline.h.md). |
| [`group_state_custom_inline.h`](group_state_custom_inline.h.md) | Stand still and play the idle clip the brain asked for by number, with a sound chosen from that number. |
| [`group_state_eat.h`](group_state_eat.h.md) | Declares the pack feeding brain, implemented in [`group_state_eat_inline.h`](group_state_eat_inline.h.md). |
| [`group_state_eat_drag.h`](group_state_eat_drag.h.md) | Declares the corpse-dragging state, implemented in [`group_state_eat_drag_inline.h`](group_state_eat_drag_inline.h.md). |
| [`group_state_eat_drag_inline.h`](group_state_eat_drag_inline.h.md) | Grip a ragdoll by an authored set of bones and haul it backward into the innermost part of the creature's territory before eating. |
| [`group_state_eat_eat.h`](group_state_eat_eat.h.md) | Declares the feeding leaf state, implemented in [`group_state_eat_eat_inline.h`](group_state_eat_eat_inline.h.md). |
| [`group_state_eat_eat_inline.h`](group_state_eat_eat_inline.h.md) | Take one bite per interval from the corpse, and stop when the meal is over, the body is out of reach, or the pack leader turns up. |
| [`group_state_eat_inline.h`](group_state_eat_inline.h.md) | The pack feeding sequence: walk to the corpse, drag it into cover if you can, play the tearing animation, eat, then withdraw and settle — with a twenty-second satiety clock governing the whole thing. |
| [`group_state_hear_danger_sound.h`](group_state_hear_danger_sound.h.md) | Declares the pack response to a frightening sound, implemented in [`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md). |
| [`group_state_hear_danger_sound_inline.h`](group_state_hear_danger_sound_inline.h.md) | A pack that hears gunfire puts its home region between itself and the sound — one member leads and the rest gather round it. |
| [`group_state_home_point_attack.h`](group_state_home_point_attack.h.md) | Declares an unused pack variant of "fall back to the home region during a fight", implemented in [`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md). |
| [`group_state_home_point_attack_inline.h`](group_state_home_point_attack_inline.h.md) | When the enemy stands somewhere a creature cannot walk, stop trying: fall back to a reserved cover spot on the enemy's side of the territory and watch the open ground. |
| [`group_state_panic.h`](group_state_panic.h.md) | Declares the pack panic brain, implemented in [`group_state_panic_inline.h`](group_state_panic_inline.h.md). |
| [`group_state_panic_inline.h`](group_state_panic_inline.h.md) | Fear, as a two-beat loop: sprint into the territory away from the enemy, stop and stare at the open ground for three seconds, repeat — unless something frightening happens during the pause. |
| [`group_state_panic_run.h`](group_state_panic_run.h.md) | Declares the pack fleeing rung, implemented in [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md). |
| [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md) | Flee *toward* the pack's territory, on the far side from the enemy — and keep fleeing until you are both far away and unobserved. |
| [`group_state_rest.h`](group_state_rest.h.md) | Declares the pack idling brain, implemented in [`group_state_rest_inline.h`](group_state_rest_inline.h.md). |
| [`group_state_rest_idle.h`](group_state_rest_idle.h.md) | Declares the wandering-within-the-territory state, implemented in [`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md). |
| [`group_state_rest_idle_inline.h`](group_state_rest_idle_inline.h.md) | Wander the territory: walk to a spot nobody else has claimed, do something idle there, walk somewhere else — and sniff the ground on some of the short walks. |
| [`group_state_rest_inline.h`](group_state_rest_inline.h.md) | The idling brain: obey a smart terrain if one is calling, stay inside your permitted region and your territory, and otherwise run a sleep/wake cycle punctuated by flavour animations. |
| [`group_state_squad_move_to_radius.h`](group_state_squad_move_to_radius.h.md) | Declares the two "close to a ring around the enemy" states, implemented in [`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md). |
| [`group_state_squad_move_to_radius_inline.h`](group_state_squad_move_to_radius_inline.h.md) | Two ways to close to a ring around the enemy: fan the pack out by squad index across an arc on the home side, or simply approach straight in. |
| [`state_adapter.h`](state_adapter.h.md) | Lets a creature state be written as a plain object with no knowledge of the state tree, by wrapping it in a node the tree understands. |
