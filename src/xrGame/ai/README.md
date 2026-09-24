# src/xrGame/ai — the creatures

Chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../SYSTEM-REQUIREMENTS.md#7-build-order):
concrete brains built on the generic machinery of
[chapter 14](../../xrAICore/README.md). Chapter 14 knows what a plan is and what a path is
and nothing about a dog. This chapter knows about the dog.

Two kinds of mind live here, and the split is the first thing to understand.

**Creatures** — everything non-human: dogs, boars, bloodsuckers, controllers, rats, crows.
A creature's mind is a **tree of states**. At every scheduled think the tree is walked from
the root, each level asking "which of my children should be running now", and exactly one
leaf ends up executing. Nothing is planned; nothing looks more than one step ahead. The
generic planner in chapter 14 is not used by a single creature in this chapter.

**Stalkers** — the human NPCs. A stalker's mind *is* the chapter-14 planner: goals,
evaluators over a world state, operators with preconditions and effects, and a search from
now to the goal. It is the only mind in the game that can work out a sequence of actions.
Everything in [`stalker/`](stalker/README.md) is about feeding that planner — what the
stalker can see, where it can stand, what it is holding — rather than about deciding
anything directly.

The trader is a third, degenerate case: a human who does not fight, whose whole behaviour
is an animation selector.

## Where it sits

After chapter 23 (the game: entities, the level, the alife simulation) and on top of
chapter 14 (navigation graphs, path search, the planner). It rests on chapter 11 for the
AI-audible side of sound, on chapter 10 for the script surface every creature exposes, and
on chapter 20 for the rigid-body layer that a dragged corpse and a thrown physics object go
through. It rests on the *shipped level data* more than any other chapter: the navigation
mesh, the per-vertex cover values and the authored patrol paths are inputs to almost every
decision made here.

Nothing depends on this chapter except the class-registration table that spawns its
entities.

## The load-bearing ideas, named once

Named here so that twenty creature twins can be terse. A twin that seems thin is thin
because its creature really is one of these ideas plus a configuration section.

### The state tree

A creature's brain is a *composite state*, and every node in it — root, branch and leaf —
satisfies one contract:

```text
INTERFACE State
  initialize()            # entered: claim what you need, stamp the start tick
  execute()               # one think's worth of behaviour
  finalize()              # left because you said you were done
  critical_finalize()     # left because somebody above you changed its mind
  check_start_conditions() -> bool    # would you accept being entered now
  check_completion()       -> bool    # are you done
  reselect_state()                    # composite only: choose a child
  setup_substates()                   # composite only: fill the chosen child's parameters
  check_force_state()                 # composite only: react before choosing
  remove_links(entity)                # an entity you referenced has gone away
```

The two exit paths are not a convenience. Everything a state claims — a reserved navigation
vertex, a frozen body, a suppressed cloak, a captured corpse — must be released on *both*,
and a state that releases on only one produces a bug that lasts for the rest of the level.
The recipe calls out the claim/release pairs in every twin that has one.

A composite's think is fixed and is the same for every creature:

```text
FUNCTION execute()                         # a composite node
  check_force_state()                      # a hook that may clear the selection
  IF no child is selected
    reselect_state()                       # must select one; a composite may not idle
  setup_substates() has already run for the chosen child
  child.execute()
  previous = current
  IF child.check_completion()
    child.finalize()
    clear the selection                    # next think re-selects from the top
```

and selection is:

```text
FUNCTION select(child_id)
  IF child_id is already current  RETURN         # re-selecting is a no-op, not a restart
  IF a child is current           current.critical_finalize()
  current = child_id
  setup_substates()                               # the parent fills the child's parameters
  current.initialize()                            # only then does the child start
```

Three consequences a rebuild must reproduce. **Parameters are filled before entry**, so a
child's entry hook can read them. **Re-selecting the running child does nothing**, which is
what lets a selector be written as an unconditional cascade re-evaluated every think without
restarting anything. And **completion is checked after execution**, so every state runs at
least once, even one that is already complete when entered.

### The selector, and the latch

The root of a creature's brain is its *state manager*, and its `reselect_state` is an
ordered cascade of tests over perception facts — always in the same order, and with no
hysteresis anywhere:

```text
enemy known → damage fresh → sound heard → corpse available → idle
```

The ordering alone is what stops the brain oscillating, because every fact it reads is a
*memory* with its own decay rather than an instantaneous reading. A rebuild that replaces
the memories with direct queries will get a brain that flickers.

Almost every "should I switch to X" test in the chapter is written with one idiom, and it is
worth naming because it appears several dozen times:

```text
FUNCTION should_switch_to(X) -> bool
  IF the previous choice was not X  RETURN X.check_start_conditions()
  RETURN NOT X.check_completion()
```

Read it as a latch. A state's start condition is about *opportunity* and its completion is
about *being finished*; asking the right one of the two is what keeps a state from being
re-entered on the think after it completed.

### State identifiers are bit-tagged

Every state in the chapter has a numeric identifier drawn from one global enumeration, and
the numbering is structural, not arbitrary: each *global* state owns one bit, and its
substates carry that bit plus a small index. So a substate identifier says which global
state it belongs to, and "is the creature attacking" is a mask test against the running leaf
rather than a walk up the tree.

The scheme has a hard limit — one bit per global state in a fixed-width identifier — and the
chapter has used most of them. A rebuild is free to use a tagged pair instead, but the
*values* leak into the script surface: a script forces a creature into a state by naming its
identifier, so the numbering is part of the frozen script contract, not an internal detail.

### The parent hands the child a parameter record

A leaf state does not query the world for its own destination. Its parent fills a small
record — the point, the navigation vertex, the action, the arrival distance, the route
rebuild interval, the acceleration profile, which sound to make and how often — and the leaf
executes it. That is why there are so few leaf states and so many composites: `move to a
point`, `look at a point` and `hold one action` between them cover most of the chapter, and
the creature's identity lives in the composites that parameterise them.

In the original the handoff is a byte copy into a pointer the child declared, which is
incidental. What survives is the *shape*: a leaf declares a parameter record type and a
parent fills one before the leaf is entered.

### What a creature is made of

Every creature is the same bag of named sub-objects, assembled by the shared base in
[`monsters/basemonster/`](monsters/basemonster/README.md). A concrete creature fills the bag
differently and answers a fixed set of virtual questions differently; it almost never adds
machinery.

- **Four memories**, each a decaying record: enemies, heard sounds, known corpses, hits
  taken. Perception is event-driven — the senses push into these — and every decision reads
  the memory rather than the sense.
- **Managers over those memories**: an enemy manager that picks *one* current enemy out of
  the enemy memory and grades its danger, and a corpse manager that picks one corpse. Memory
  records what happened; a manager decides what to care about. This split is why a creature's
  selector can be a flat cascade.
- **Four control channels** — animation, movement, path building, direction — arbitrated by
  a control manager that decides which component owns each channel at a time. A behaviour
  state does not move the creature; it *requests* an action and writes path parameters, and
  the channels resolve the request after the brain has finished.
- **Place and coordination**: the cover manager (a query over the level's per-vertex cover
  values), the home object (an authored territory with three nested radii), the anomaly
  detector, the footstep manager, and the pack the creature belongs to.
- **Body and condition**: melee-reach checking, morale, the physics support, four auras that
  radiate an influence at nearby entities, skin armour, and a bone-keyed critical-wound map.

### The managers a state reaches for

Four objects appear in creature states constantly, and each answers one question:

| Manager | Question it answers | Where the answer comes from |
|---|---|---|
| **cover** | where can I stand so he cannot see me | the level's per-vertex cover values, shipped prebuilt |
| **home** | am I inside my territory, and where in it should I go | an authored region with an inner, middle and outer radius |
| **anomaly detector** | is there something ahead that will hurt me | zones the creature has touched or been told about |
| **pack (squad)** | what is everyone else doing, and is this spot taken | a registry of packs keyed by group and team |

The pack is the only one that mutates: members *claim* navigation vertices and corpses from
it and must release them on every exit. The claims are advisory — nothing physically stops
two creatures standing on one spot — but they are the entire reason a pack spreads out
instead of stacking on the single best cover point.

### Most creatures are data

This is the claim that keeps the chapter from reading as twenty designs. Strip away the
configuration and the model, and the shipped creatures are:

- **the same brain**: the same ordered selector over the same five global states, drawn from
  the shared library in [`monsters/states/`](monsters/states/README.md);
- **plus zero, one or two distinctive abilities**, each a registered extra state and one
  extra test at the top of the selector.

Boar, cat, flesh, fracture, snork, tushkano and zombie add nothing but numbers and one
signature move. The psy dog's entire difference from the pseudodog is *one* registered state
and *one* condition. Bloodsucker, burer, controller, poltergeist, chimera and the pseudo
giant are the genuinely elaborate ones, and even they are the shared brain with an ability
tree spliced in.

What "the numbers" means concretely: an entity is a class identifier plus a **section** —
the configuration section supplying every tuned value. Movement speeds, attack reach,
morale, panic threshold, sound delays, route rebuild cadences, aura strengths, the
animation set, the pack separation, the effector look. A rebuilder who reimplements the
code faithfully and invents the numbers gets creatures that are recognisably wrong, because
the numbers are where the tuning went. They ship in the game data, not here.

The parts that are *not* data are worth naming, because they are inconsistent: most of the
timing constants inside behaviour states — how long a stalking creature camps before
relocating, how long a pack stays satiated, how wide a retreat annulus is — are fixed in
code with no comment. The twins flag these individually.

### The stalker is the elaborate one

Everything above is about creatures. A stalker is different in kind.

Its brain is the chapter-14 planner. Goals are proposed by *motivations* weighted each
cycle; the planner searches over operators whose preconditions are expressed as evaluators
over a symbolic world state; the resulting plan is executed one operator at a time and
abandoned the moment a property it relied on moves. That is the only mind in the game that
can answer "to do this I must first do that".

What lives in [`stalker/`](stalker/README.md) is not the planner but its senses and hands:
weapon selection and aiming with its own dispersion model, cover evaluation against a
remembered enemy position, perception that must distinguish a friend from a stranger from
an enemy, the script-entity path by which a level's scripts drive a stalker directly, and
the event plumbing that turns a hit or a death into something the planner's evaluators can
read. It is also where the game's *social* model — factions, reputation, who may be talked
to — reaches the AI.

A rebuild can substitute a behaviour tree for the planner and get something that plays
similarly in a firefight and noticeably worse everywhere else, because the sequences the
planner finds — put the weapon away, walk over, pick that up, take cover, reload — are what
makes a stalker look like a person with an errand.

### Packs, and why one creature's brain is different

One creature family — the dog — has its brain written against the *pack* rather than the
individual, in [`monsters/group_states/`](monsters/group_states/README.md). A dog's attack,
rest, panic, feeding and sound responses are all pack states: they read the pack's shared
judgement ("is our territory threatened"), they fan out by pack index, and they claim spots
from each other. No other creature in the chapter does this, so the directory reads as a
parallel universe of the shared state library. It is worth reading precisely because the
difference between the two libraries is a clean statement of what coordination costs.

### The rat is not like the others

[`monsters/rats/`](monsters/rats/README.md) predates the rest and shares almost nothing with
it: its own state enumeration, its own transition machinery, its own flocking. The recipe
keeps it as its own thing rather than pretending it is an instance of the pattern above.

## What is in this directory

| Twin | Role |
|---|---|
| [`ai_monsters_anims.h`](ai_monsters_anims.h.md) | Loads a creature's animation set by *naming convention*, so a new creature's motions can be added in data without touching code. |
| [`ai_monsters_misc.cpp`](ai_monsters_misc.cpp.md) | Decides, once per group per refresh period, whether a squad of humans should attack, hold or retreat, by estimating its collective odds against the enemies it can see. |
| [`ai_monsters_misc.h`](ai_monsters_misc.h.md) | Declares the group tactical vote, and defines the transition vocabulary of the engine's older, stack-based creature state machine. |
| [`position_prediction.h`](position_prediction.h.md) | Leads a moving target: estimate where the enemy will be by the time I get there. Compiled into the game and included by nothing. |
| [`weighted_random.cpp`](weighted_random.cpp.md) | Draws from a one-, two- or three-point piecewise-linear distribution by rejection sampling inside a trapezoid. Never called. |
| [`weighted_random.h`](weighted_random.h.md) | Declares a sampler that draws from a piecewise-linear distribution over one, two or three points. Included once, used nowhere. |

## Subdirectories

| Directory | Contents |
|---|---|
| [`monsters/`](monsters/README.md) | The creature layer: the state machinery, the shared sub-objects, the control channels, the abilities, and every concrete creature |
| [`monsters/basemonster/`](monsters/basemonster/README.md) | The shared creature base — the bag of sub-objects every creature is assembled from |
| [`monsters/states/`](monsters/states/README.md) | The shared state library: the generic rest, eat, attack, panic, hit and sound behaviours every creature draws on |
| [`monsters/group_states/`](monsters/group_states/README.md) | The pack-aware parallel library, used by the dog |
| [`monsters/bloodsucker/`](monsters/bloodsucker/README.md) | The bloodsucker: cloaking, feeding, and a great deal of unreachable combat |
| [`monsters/boar/`](monsters/boar/README.md) | The boar: the chapter's baseline creature |
| [`monsters/burer/`](monsters/burer/README.md) | The burer: telekinesis, a gravity blast, and a shield |
| [`monsters/cat/`](monsters/cat/README.md) | The cat |
| [`monsters/chimera/`](monsters/chimera/README.md) | The chimera: a leaping predator with a hunting tree that does not compile |
| [`monsters/controller/`](monsters/controller/README.md) | The controller: a fragile ranged creature that fights with psychic attacks and mind control |
| [`monsters/dog/`](monsters/dog/README.md) | The dog: the only creature whose brain is written against its pack |
| [`monsters/flesh/`](monsters/flesh/README.md) | The flesh |
| [`monsters/fracture/`](monsters/fracture/README.md) | The fracture |
| [`monsters/poltergeist/`](monsters/poltergeist/README.md) | The poltergeist: an invisible creature that fights entirely through thrown objects or flame |
| [`monsters/pseudodog/`](monsters/pseudodog/README.md) | The pseudodog and the psy dog, which fights by sending out illusions of itself |
| [`monsters/pseudogigant/`](monsters/pseudogigant/README.md) | The pseudo giant: a slow heavy creature whose footfalls shake the camera |
| [`monsters/rats/`](monsters/rats/README.md) | The rat: an older, separate architecture kept alive alongside the rest |
| [`monsters/snork/`](monsters/snork/README.md) | The snork: a leaping ambusher |
| [`monsters/tushkano/`](monsters/tushkano/README.md) | The tushkano: the simplest creature in the game |
| [`monsters/zombie/`](monsters/zombie/README.md) | The zombie: a creature defined by refusing to die |
| [`crow/`](crow/README.md) | The crow: ambient flying decoration that can be shot down |
| [`phantom/`](phantom/README.md) | The phantom: an apparition that flies at the player and bursts |
| [`stalker/`](stalker/README.md) | The human NPC: the only mind in the game that plans |
| [`trader/`](trader/README.md) | The trader: a human who never fights, driven by an animation selector |

## What could not be recovered

Collected from across the chapter so the root README's honesty section can absorb it. Items
specific to one creature are repeated in that creature's own directory README.

- **A large block of the bloodsucker's combat is deliberately switched off.** Its own attack
  composite — with the back-approach, the wounded withdrawal and the reactive stalking loop
  it alone reaches — is replaced at registration by the generic attack composite, with the
  original line commented out beside it. Whether this was a temporary measure or a shipped
  decision is not recoverable from the source, and the behaviour is substantial enough that a
  rebuild must choose consciously.
- **The chimera's hunting tree cannot compile.** One of its state headers is a verbatim copy
  of its sibling, so the same method is defined twice in the same translation unit. Nothing
  instantiates the tree, which is the only reason the build succeeds. What the second state
  was meant to be is not recoverable.
- **The chimera's threaten tree is registered nowhere**, and the global state it was written
  for is never selected by any creature.
- **One of the burer's melee states sits in its dispatch table but is never selected** by any
  branch of its selector.
- **The controller's psychic-fire state is declared, implemented and included by its attack
  composite, but never instantiated.** The attack itself still happens, driven from the
  creature's ability rather than from a behaviour state.
- **Several behaviour timings are fixed in code with no derivation**: fifteen seconds before
  a stalking bloodsucker relocates, twenty seconds of pack satiety after a meal, three
  seconds of circling, a ten-second and a two-second perception window in the psy dog's aura,
  a two-beat oscillation in the feeding screen effect. Each is reproducible and each is
  clearly deliberate; none is explained anywhere, and none is in configuration.
- **The state identifier numbering is a script contract** and must be verified against a
  running original rather than trusted from the source. Scripts force creatures into states
  by identifier, so any renumbering silently changes what shipped scripts do.
- **Half the files at the top of this chapter are unused.** The target-leading predictor is
  compiled and included by nothing, and the weighted sampler is included once and called
  nowhere. Both are complete and both look like they were meant for the creature layer. A
  third defines the transition vocabulary of the engine's older, stack-based creature
  machine, which now survives only in the rat.
- **Whether a state may be entered twice in one think** is not stated anywhere; the tree's
  contract makes it impossible by construction, but several selectors are written as though
  the author was not sure.
