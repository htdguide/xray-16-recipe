# src/xrGame/ai/monsters/zombie — the zombie

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Shared machinery: the [chapter opener](../../README.md) and the [creature layer](../README.md).

The zombie is a creature defined by one mechanic: **it does not stay down**. Everything else
about it is the shared brain with two states missing.

## What is actually its own

**Feigned death.** Below a configured health threshold the zombie falls, plays one of four
collapse sequences, lies still for a while, and gets up again — a configured number of times
before a fall is finally real. The count and the threshold are data; the four sequences are
authored clip triples chosen at random.

This is not a state in the behaviour tree. It is handled in the damage path and in the
scheduled update, and the brain is simply *suppressed* while it runs: the selector returns
immediately whenever the creature's triple-animation channel is active. That placement is
the load-bearing decision. Feigning death has to survive a brain that would otherwise choose
a state and issue movement over the top of the collapse, and making it a state would have
required every other state to know about it.

**An attack run of its own**, substituted into the shared attack composite in place of the
generic one — the shambling approach that will not break off.

**Bone manipulation on the spine and head**, so the zombie tracks with its upper body
without turning, and **aiming at its centre rather than its head**, which is why it is easy
to hit and looks wrong to aim at.

**Three states it does not have.** No panic, no dangerous-sound response, and no reaction to
being hit: both kinds of heard sound route to the same interested response, nothing
frightens a zombie and nothing interrupts it. That is the whole reason it reads as
relentless, and it is expressed as an *absence* in the registered state set rather than as a
rule anywhere. A rebuilder reading only the selector will see a short cascade and miss that
the shortness is the design.

## What could not be recovered

- The count of four collapse sequences is a compile-time constant with no derivation; the
  number of *usable* ones depends on the model having four authored triples.
- The resurrection delay is stored on the creature but the value it is set from is not
  traceable to a configuration key in this directory.

## Twins

| Twin | Role |
|---|---|
| [`zombie.cpp`](zombie.cpp.md) | Implements the zombie, whose one idea is a feigned death that is cheaper to survive each time until it runs out. |
| [`zombie.h`](zombie.h.md) | Declares the zombie — the creature whose defining trick is that it plays dead and gets back up. |
| [`zombie_script.cpp`](zombie_script.cpp.md) | Exports the zombie's class identity to the script layer. |
| [`zombie_state_attack_run.h`](zombie_state_attack_run.h.md) | Declares the zombie's approach: the only creature-specific leaf state in its behaviour tree. |
| [`zombie_state_attack_run_inline.h`](zombie_state_attack_run_inline.h.md) | Implements the shamble: walk at the enemy, re-pathing more lazily the further away he is, and break into a run only after being shot. |
| [`zombie_state_manager.cpp`](zombie_state_manager.cpp.md) | The zombie's mind, defined as much by what it omits as by what it has: no panic, no flight, no reaction to being hit. |
| [`zombie_state_manager.h`](zombie_state_manager.h.md) | Declares the zombie's behaviour tree root. |
