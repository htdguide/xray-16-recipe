# src/xrGame/ai/monsters/rats/ai_rat_fsm.cpp

> The rat's previous brain: a switch-driven loop that ran until a state declared itself settled. It is excluded from the build, it no longer compiles, and it is superseded by the state stack — but it is the only readable record of what several of the live predicates were originally for.

**Needs** — [`ai_rat.h`](ai_rat.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: dead code; nothing to implement

## Purpose

**Do not reproduce this file.** It is commented out of the build list, it defines a think step
whose name collides with the live one in
[`ai_rat_behaviour.cpp`](ai_rat_behaviour.cpp.md), and it references at least a dozen
identifiers that no longer exist — state names that were renamed, fields that moved, helper
routines that were replaced. Re-enabling it would not compile, let alone run.

It is worth one page for two reasons. First, the *shape* of brain it represents is a real
alternative to the stack the rat now has, and the reason it was abandoned is legible. Second,
several predicates in [`rat_state_switch.cpp`](rat_state_switch.cpp.md) read as arbitrary until
you see the conditions they were extracted from, which are here.

## The brain shape that was replaced

```text
FUNCTION Think()          # the dead version
  update_morale()
  update_anchor()
  settled = false
  REPEAT
    previous_state = current_state
    run the routine for current_state        # which may assign current_state and clear `settled`
    state_changed = (previous_state != current_state)
  UNTIL settled
```

Each state routine set a "stop thinking" flag on entry, and the transition macros cleared it
again when they changed state — so **a tick kept re-running state routines until one of them
chose to stay put**. That gave transitions with no frame of latency: a rat that acquired an
enemy while wandering entered the approach and ran one tick of it immediately.

The live brain runs each state exactly once per tick and lets it push or pop, which costs a tick
of latency on every transition and buys three things the loop could not give: a genuine
"return to what I was doing" (the loop had only a single previous-state slot), no possibility of
a transition cycle spinning the tick forever, and states as objects that can be registered and
replaced rather than switch arms.

There is a third shape visible in the fossil: the transition macros came in *four* varieties —
change state and re-run this tick, change state and let it start next tick, return to the
previous state, and return to the previous state next tick. The live brain expresses all four
with push, pop and one-per-tick execution.

## What the fossil records that the live code does not

- **The conditions the live predicates were carved out of.** `switch_to_free_recoil`,
  `switch_if_no_enemy` and `switch_to_attack_melee` in
  [`rat_state_switch.cpp`](rat_state_switch.cpp.md) are each a compound condition lifted
  verbatim from one of these routines, which is why they read as unmotivated conjunctions. The
  originals here show what they meant: *heard something hostile that is not the corpse I am
  eating and have no enemy*; *the enemy is gone, dead, or long unseen with nothing frightening
  still audible*; *the enemy is within pursuit range of the nest, or I have strayed too far from
  it*.
- **A combat lottery that is still live but hard to read.** The approach and retreat states both
  called a shared group-level decision routine with the rat's team, squad and group, an
  authored success probability and a refresh rate, to pick between continuing and retreating.
  Its purpose is to make a *group* commit or break together rather than each rat deciding alone.
  The live brain still calls it, from `get_state` in
  [`rat_state_switch.cpp`](rat_state_switch.cpp.md).
- **A "no enemy in range" retreat** that the live brain reaches differently.
- **The corpse-eating state's inner logic**, which was moved almost unchanged into
  [`rat_state_activation.cpp`](rat_state_activation.cpp.md) — close enough that the two can be
  diffed, which is the quickest way to see what the port changed and what it did not.

## What is broken in it

Recorded so nobody tries to revive it: it names the death state, the bite state and the approach
state by identifiers that were renamed; it calls a fire setter, a position integrator, a
movement-type setter and a spawn-anchor updater that were all renamed or replaced; it reads
morale and last-update fields under their old names; and it defines a think step that would
collide with the live one at link time. The build list comments it out rather than deleting it,
which is how it survived.
