# src/xrGame/ai/monsters/group_states/group_state_panic_inline.h

> Fear, as a two-beat loop: sprint into the territory away from the enemy, stop and stare at the
> open ground for three seconds, repeat — unless something frightening happens during the pause.

**Needs** — [`group_state_panic.h`](group_state_panic.h.md) · [`group_state_panic_run.h`](group_state_panic_run.h.md) · [`../states/state_look_unprotected_area.h`](../states/state_look_unprotected_area.h.md) · [`../states/monster_state_home_point_attack.h`](../states/monster_state_home_point_attack.h.md) · [`../states/state_data.h`](../states/state_data.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`group_state_panic.h`](group_state_panic.h.md)
**Tier floor** — T3: a two-rung alternation with two pre-emptive interrupts

## Purpose

The pack counterpart of the generic panic state. It differs in exactly one substitution — the
fleeing rung is the pack version, which runs *toward the pack's territory* rather than simply away
(see [`group_state_panic_run_inline.h`](group_state_panic_run_inline.h.md)) — and is otherwise the
same two-beat loop.

The loop's shape is what makes panic readable. A creature that only ran would disappear over the
horizon; one that only cowered would be a target. Alternating a sprint with a three-second
frightened pause is what produces the recognisable bolting-and-checking behaviour, and the
interrupts below are what keep the pause from becoming a liability.

## State

`Stateless.` Three registered substates: flee (pack version), face the open ground, and go home
(the generic creature version).

## `reselect_state`

**Contract** — territorial defence first, then alternate.

```text
FUNCTION reselect_state()
  IF the go-home substate will accept      # we are outside our territory, or the enemy is unreachable
    select go_home; RETURN

  IF the previous rung was flee  -> face the open ground
  ELSE                           -> flee
```

**Notes** — the go-home rung is the same one the attack brain uses, which means a panicking
creature and an attacking creature both prioritise getting back inside their region over the
behaviour they are in. Territory outranks both fear and aggression, throughout this chapter.

## `setup_substates`

**Contract** — parameterise only the facing rung: stand idle for 3 seconds with the *scared*
animation flag raised, playing the panic sound at the section's attack-sound delay.

**Notes** — the scared flag is what selects each creature's own frightened stance from its
animation table (see [`../flesh/flesh.cpp`](../flesh/flesh.cpp.md), where "look around" is mapped
to the scared clip). The three seconds is authored here, in code; the *sound* delay is authored
per section. That split is typical of the chapter: the timing of the behaviour is code, the
character of the noise is data.

## `check_force_state`

**Contract** — evaluated before the ordinary selection each tick. While the creature is in the
facing rung, two observations cut the pause short and send it straight back to fleeing.

```text
FUNCTION check_force_state()
  IF the active rung is "face the open ground"
    IF the enemy was seen this very tick                  -> flee
    IF we were hit within the last five seconds           -> flee
```

**Notes** — this is the whole reason panic is a composite rather than a pair of states, and it is
the file's load-bearing decision. Without it, a creature that stops to cower is committed to three
seconds of standing still while being shot — which reads as broken. With it, the pause exists only
while nothing is happening, and the creature resumes running the instant it sees its pursuer or
takes a round.

The two tests are deliberately asymmetric in how fresh they demand their evidence. Sight must be
*this tick* — the creature must be looking at the enemy right now, not remembering it. A hit
counts for five full seconds, because being shot from an unseen direction is exactly the case
where the creature has no sighting to go on and should still be running.

Being a *force* hook rather than part of the ordinary selection matters: it runs while a rung is
still active and has not reported completion, so it can pre-empt. The ordinary selector only runs
between rungs.
