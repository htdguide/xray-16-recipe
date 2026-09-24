# src/xrGame/ai/monsters/basemonster — the shared creature base

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../../README.md) and the [creature layer](../README.md) first.

Twelve files holding one class: **what every non-human creature in the game is**. Every
creature in the sibling directories is this plus a configuration section, an animation table,
and at most two abilities.

## The central idea

A creature is **not a class with behaviour in it**. It is a bag of named sub-objects plus a
set of virtual questions, and a concrete creature is built by filling the bag differently and
answering the questions differently. The declaration's member list is therefore the
architecture of the whole chapter, and reading it is worth more than reading any individual
creature.

The bag holds four memories, two reducers over them, four control channels behind an
arbitrating bus, an ability roster, the place objects (cover, home, anomaly detector,
footsteps, pack), the body (physics, morale, melee reach, four auras, armour, a critical-wound
bone map), a brain the base never constructs, and a shared settings block.

The questions are ten *ability declarations* — can this creature turn invisible, drag corpses,
attack with psi, cause an earthquake, jump, feel at a distance, attack on the run,
rotation-jump, jump over physics objects, does it need pitch correction — each answered with a
constant. They are not capabilities the base implements. They are **declarations other
machinery branches on**, which is why they are questions and not fields in the settings block.

## The think, in order, and why the order is the design

```text
FUNCTION think()
  IF not alive OR being destroyed  RETURN
  the creature's own per-think hook          # empty by default
  the animation layer's per-think preparation
  update_memory()                            # perception: what do I know now
  pack.update(self)                          # coordination: if I lead, decide for everyone
  update_state_machine()                     # decision, and its consequences
```

Perception is refreshed **before** the pack coordinates, and the pack coordinates **before**
any member's brain runs. That is what makes the pack's commands consistent with what its
members can actually see, and it works because coordination runs only on the leader's think:
by the time a follower thinks, the leader's pass has already happened.

Then, after the brain:

```text
FUNCTION update_state_machine()
  brain.update()                    # picks and runs a state; sets an abstract action
  derive the run-turn flags from the chosen state
  translate the abstract action into movement parameters
  report my goal up to the pack
```

The brain's *only* output is an abstract action plus whatever it wrote into the control
channels. Turning that into movement parameters happens after, once, in one place — which is
why no behaviour state anywhere in the chapter has to know about gaits or velocity masks.

## Bringing a creature up

Four passes, in this order, and the order is load-bearing:

```text
reload(section)   # sounds, creature type, home — before Load
Load(section)     # the base's parameters and sub-object construction
  ... the concrete creature's own load runs here ...
PostLoad(section) # parameters that need the concrete creature's data to exist
reinit()          # every mutable field back to its spawn value
```

The third pass exists because two of its jobs need the creature's *velocity table*, which the
concrete creature fills. The fourth is the most dangerous routine in the chapter: a creature is
reinitialised on spawn, on crossing back into the loaded level from the offline simulation, and
when its restrictions change — and anything it forgets to reset is state leaking across a save.

**Settings are shared by content, not by section name.** The tuned block a creature reads is
reference-counted and shared between every creature loaded from the same section, keyed by a
checksum of the block's bytes. Two sections with identical numbers share one block.

## Two derived facts the whole chapter branches on

The perception refresh ends by deriving two summary flags — *heard something interesting* and
*heard something dangerous* — from the sound memory. Almost every creature selector in the
chapter reads exactly these two booleans rather than the memory behind them. A rebuilder who
exposes the memory directly will find the selectors much harder to write.

## Where the seams are

The creature base is where chapter 24 touches the rest of the engine: the rigid-body layer
(corpse capture, ragdolls, thrown objects), the collision database (perception rays, cover
tests), the audio device (the sound bank a creature loads by scheme), and the script binding
layer (six kinds of script-assignable action, the follow offset, and the cross-level path
decision). The individual twins name each.

## What could not be recovered

- **The header carries a block comment reading "Kill From Here"** above a run of public mutable
  flags — damaged, angry, growling, aggressive, asleep, turning left while running, turning
  right while running — that are read and written from everywhere. The comment is a complaint,
  not a boundary. A rebuild should make each of them derived or owned; the twins say where each
  is decided.
- The four memory retention periods (twenty seconds for enemies, sounds and corpses; fifty for
  hits) are set at construction rather than read from data, unlike almost everything else about
  a creature.
- Health is read from configuration as an integer while every other condition value is
  fractional. Nothing explains the inconsistency.

## Twins

| Twin | Role |
|---|---|
| [`base_monster.cpp`](base_monster.cpp.md) | The creature base's core: assembly, the two update paths, damage, death, team membership, sound pacing, and the translation from an abstract action into movement parameters. |
| [`base_monster.h`](base_monster.h.md) | Declares the creature base — the assembly of managers, memories and control channels that every non-human creature is made of, and the virtual questions each creature answers differently. |
| [`base_monster_anim.cpp`](base_monster_anim.cpp.md) | The animation-selection entry point, which a creature answers by handing the question straight to its animation control channel. |
| [`base_monster_debug.cpp`](base_monster_debug.cpp.md) | The creature's introspection: a full tree of everything it currently knows, believes and is doing, built only in a development build. |
| [`base_monster_feel.cpp`](base_monster_feel.cpp.md) | Perception in, damage out: what a creature hears and what it may see, and everything that happens on the player's screen when a creature hits him. |
| [`base_monster_inline.h`](base_monster_inline.h.md) | The panic-threshold override pair, separated out so scripts can raise a creature's nerve and put it back. |
| [`base_monster_misc.cpp`](base_monster_misc.cpp.md) | The perception refresh: advance every memory, let the managers choose what matters, and derive the two summary flags the whole behaviour layer branches on. |
| [`base_monster_net.cpp`](base_monster_net.cpp.md) | The creature's wire form and its save relevance. |
| [`base_monster_path.cpp`](base_monster_path.cpp.md) | How a creature turns to face a point, and the four cover queries every creature's states are written in terms of. |
| [`base_monster_script.cpp`](base_monster_script.cpp.md) | How a Lua script drives a creature: the six action kinds it can assign, the follow-the-leader offset, and the cross-level path decision. |
| [`base_monster_startup.cpp`](base_monster_startup.cpp.md) | Everything a creature reads from its configuration section, the sound bank it loads, and the reset that must leave it exactly as a fresh spawn. |
| [`base_monster_think.cpp`](base_monster_think.cpp.md) | The creature's think: refresh perception, coordinate the pack, run the state machine, translate the result into movement, and report back up to the pack. |
