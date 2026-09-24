# src/xrGame/ai/monsters/controller/controller_state_manager.cpp

> The controller's top-level brain: seven global states, a strict priority ordering over them,
> and one rule about when an external ability may interrupt.

**Needs** — [`controller_state_manager.h`](controller_state_manager.h.md) · [`controller.h`](controller.h.md) · [`controller_state_attack.h`](controller_state_attack.h.md) · [`controller_state_attack_hide.h`](controller_state_attack_hide.h.md) · [`../monster_state_manager.h`](../monster_state_manager.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`controller_state_manager.h`](controller_state_manager.h.md)
**Tier floor** — T3: a priority selector over perception facts, evaluated once per creature update

## Purpose

Every creature's brain is the same machine — a state that owns substates and re-picks one each
update — and every creature differs in *which* substates it owns and *in what order* it prefers
them. This file is the controller's answer to both questions. It is the smallest interesting
brain in the chapter, because the controller is a solitary creature: no squad, no pack states, no
home-region tactics beyond what the shared attack composite supplies.

## State

The manager owns no data of its own. It holds the seven registered global states and the two
fields every composite state has: which state is active and which was active last update.

Registered states, with the implementation chosen for each:

| Global state | Implementation | Note |
|---|---|---|
| rest | generic | |
| panic | generic | registered; never selected automatically |
| heard an interesting sound | generic | |
| heard a dangerous sound | generic | |
| was hit | generic | |
| attack | [controller-specific](controller_state_attack_inline.h.md) | |
| eat | generic | |
| custom | [controller retreat](controller_state_attack_hide_inline.h.md) | reachable only by script |

Two of these are never chosen by the selector below: **panic** and **custom**. They are
registered so the script layer can force the creature into them — forcing a state is exactly
"select this registered state and execute it", so registration *is* the script-visible surface. A
rebuild that prunes unreachable states by reading the selector alone will silently break shipped
scripts.

## `CStateManagerController`

**Contract** — construct with the creature it drives; register the seven states. Re-init puts the
creature into its idle demeanour in addition to the base reset. Each update, choose exactly one
global state, switch into it if it differs from the active one, execute it, and record it as the
previous state.

**Invariants** — exactly one state is selected on every update; there is no path that leaves the
brain without an active state. The previous-state record is updated last, after execution, so any
state that inspects it during execution sees the state it followed, not itself.

```text
FUNCTION update()
  IF an enemy is known            -> attack
  ELSE IF hit memory is fresh     -> was_hit
  ELSE IF heard an interesting sound -> hear_interesting
  ELSE IF heard a dangerous sound -> hear_danger
  ELSE IF a usable corpse is near AND the eat state will accept -> eat
  ELSE                            -> rest

  switch_to(chosen)               # no-op when already active
  active_state.execute()
  previous = chosen
```

## `check_control_start_conditions`

**Contract** — answers whether an externally driven ability may take over the creature right now.
The controller admits exactly one: the anti-aim evasion, and only while the attack composite is
in its *run at the enemy* substate. Every other ability is refused unconditionally.

**Notes** — this is a narrow, deliberate answer and it is the one place the controller's brain
reaches into its own substate. The reasoning is visible in play: anti-aim makes the creature
dodge while closing, and dodging while standing still (mid-strike, mid-retreat, eating) would
either cancel a committed action or look like a glitch. The default for a creature that does not
override this is "ask the active substate", so returning a flat refusal here is a *narrowing*, not
a default.

The selector has no hysteresis of any kind: every fact it reads is a memory with its own decay,
and the ordering alone is what keeps the brain from oscillating. That is the shared pattern —
combat beats damage beats sound beats appetite beats idling — and the per-creature files in this
chapter differ mainly by what they insert into it.

Two constants sit unused above the selector: a five-second "enemy hidden" interval and a
ten-unit distance, evidently the parameters of a find-enemy branch that was planned here and lives
instead inside the shared attack composite. Nothing reads them.
