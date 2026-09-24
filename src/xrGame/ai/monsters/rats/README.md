# src/xrGame/ai/monsters/rats — the rat

Part of chapter 24 of [`SYSTEM-REQUIREMENTS.md`](../../../../../SYSTEM-REQUIREMENTS.md#7-build-order).
Read the [chapter opener](../../README.md) for the machinery every other creature shares —
and then set it aside, because the rat uses almost none of it.

The rat is the oldest AI in the engine and it was never brought forward. It does not derive
from the creature base, so it has no control bus, no morale object, no home object, no
footstep manager, no state tree, no parameter records and no shared behaviour library. It has
its own hand-rolled version of each, and a rebuilder must decide up front whether to port the
rat as it is or to reimplement it as an ordinary creature — the two produce noticeably
different animals.

## What it is instead

**It is simultaneously a creature and an inventory item.** The rat is the only entity in the
chapter that is also *edible loot*: a dead rat can be picked up, carried and eaten. That
second identity is why the class hand-writes its own network form, its own physics-shell
handoff and its own usefulness test instead of inheriting them.

**Its brain is a push-down automaton, not a tree.** A stack of state identifiers, twelve
registered states, and a three-method state interface — enter, run, leave. No nesting, no
completion test, no re-selection hook, no parameter blob. Transitions are decided inside each
state's body as a hand-ordered cascade of predicate calls, and a transition is either "push",
"pop", or "pop then push".

Note the shape of the decomposition, because it is unlike anything else in the chapter: one
conceptual state is split across *three* files by **kind of operation** rather than by state —
the predicates in one, the entry hooks in another, the per-tick bodies in a third — with the
transition wiring in a fourth file that lives outside this directory entirely.

**Its locomotion runs at frame rate, not at think rate.** The states never move the rat. They
write a goal direction and a speed, and a separate integrator chases those on the render tick:
a damped pitch controller, a smoothed heading delta, terrain-following from the navigation
contour, and a probe one frame ahead that rejects any step leaving the navigable area — on
rejection, *every* mutated field is rolled back and the rat is marked stuck, flipping around
after half a second. The shared path-following machinery is explicitly disabled.

## Group behaviour, which is not flocking

There are two separate and independently maintained group abstractions, and neither is a
steering model.

**A population quota.** The colony keeps counts of how many members are alive, how many are
"active" and how many are standing still, and a rat is admitted to the active pool only while
a configured percentage allows. Active rats tick fast and move; the rest idle on a longer
scheduler period. That is the engine's answer to "a swarm is expensive": most of the colony is
asleep at any moment. The counters live on the shared group object with a note saying they
exist for rats only and should be removed.

**Angular slot assignment.** Attacking rats are stamped with an index by distance to the
enemy, and each aims not at the enemy but at a point on a half-metre ring around him, at its
own index times a full turn divided by the pack size, measured from the leader's bearing. That
is an *encirclement pattern*, computed once per rat from an index — not separation, cohesion
or alignment, which are never computed at all.

**The colony migrates as one body.** Only the leader picks a new home vertex on the coarse
cross-level graph, every minute or two; everyone else copies the leader's home verbatim and
force-wakes when it changes. Home is what the "come back" and "stop chasing" radii are measured
against, so the whole colony drifts across the map together and drags its members back from
chases.

Leadership is computed *twice, from two different registries*, and nothing keeps the two in
agreement.

## Perception, damage and sound

Perception is two filters on the inherited memory system: any non-living or non-teammate
entity is worth seeing — dead teammates stay visible because they are food — and a heard
sound above a threshold both overwrites the remembered sound and applies an immediate morale
shock, graded by whether it was a death, a shot with no enemy in mind, or an attack.
Gunfire is forced to maximum loudness regardless of distance. Touch is ignored entirely.

Melee is a *periodic hitscan* driven off an action flag rather than off an animation frame:
while the rat is in firing mode and its interval has elapsed, a hit event is sent. There is no
reach test at that layer; reach lives in the transition predicates.

The sound vocabulary carries **eviction masks** rather than priorities: each utterance names
which currently-playing sounds it cancels. Dying and being injured cancel everything;
attacking and eating share a bit and are therefore mutually exclusive; the voice squeak
coexists with both — but cancels *itself*, which is the cause of one of the defects below.

## Where the numbers come from

Unusually, from **two** sources. Timings, distances, angular speeds, the wander parameters and
the colony quotas come from the configuration section. But the per-instance combat and
perception values — eye arc and range, the three speeds, the pursuit and home radii, all seven
morale constants, hit power and interval, attack reach, attack cone and success probability —
come from the **spawn record**, edited per rat in the level editor, with defaults baked into
the record's constructor. A rebuilder must keep both channels; reading everything from the
section will silently ignore how a level was authored.

Optionally, a rat can be given an authored patrol path in its spawn record.

## What could not be recovered

The rat carries more dead and broken code than the rest of the chapter combined. These are
behavioural, not cosmetic.

- **The death state is registered and never entered.** The code that retires a rat's corpse
  once its food is exhausted is unreachable. Dead rats keep running whatever state they were
  in — each of which begins by returning immediately because the rat is dead — forever.
- **The whole passive half of the design is unreachable.** The resting state is registered and
  never pushed, so the standing-still quota is always zero, its configuration key has no
  effect, one of the two idle animations is never played, and the death-animation bias that
  depends on it is never taken. "Deactivating" a rat therefore only slows its tick rate; it
  keeps wandering.
- **Morale dispatches on a vestigial state variable** left over from an earlier machine, which
  now only ever holds one of three values. Every combat branch of morale regeneration is
  unreachable, so a rat's morale always relaxes toward its resting value and never accumulates
  under fire — which is precisely the variable that gates panic.
- **The usefulness test looks inverted**, refusing every candidate item while the rat is
  alive. Since item memory only updates while alive, no corpse is ever selected, so the entire
  eat-corpse behaviour and its three configuration keys are dead.
- **The pursuit state is reachable from nowhere.** Its only transition is guarded by a
  condition the enclosing early return has already made false.
- **Two flags that choose the patrol steering mode are declared and never assigned anywhere.**
  Patrolling rats steer on uninitialised values.
- **Two more values are read before first assignment** on a rat's very first think, and a third
  is left unassigned when no navigation vertex type matches.
- **Reinitialisation leaks the previous brain** and its twelve states.
- **A goal-timer setter ignores its argument** and writes zero, and **a morale broadcast
  ignores its radius** and reaches the whole colony — the radius is loaded from configuration
  and passed purely for show.
- **The voice squeak almost never plays.** Its eviction mask matches itself, so every execution
  of the state cancels the pending squeak and re-rolls a fresh delay; it fires only if two
  consecutive executions of that state are further apart than the rolled interval, which
  depends on the scheduler period rather than on anything intentional.
- **The original state machine survives as a file excluded from both build systems.** It
  references nine members the class no longer has, so it could not be re-enabled without
  repair. Read it as history, not as an alternative.
- **The flocking that the file names promise was designed twice and shipped zero times.** A
  steering manager is declared on the class, set to nothing, and the three lines that would
  populate it with cohesion, separation and alignment are commented out; those three classes
  exist only as headers with no implementations. A *working* steering library does exist
  elsewhere in the engine and is used by creature packs — not by rats.
- A long tail of unread members, write-only fields, declared-but-undefined methods, unguarded
  normalisations that can produce invalid directions, and configuration keys loaded into
  members nothing reads. The individual twins record them.

## Twins

| Twin | Role |
|---|---|
| [`ai_rat.cpp`](ai_rat.cpp.md) | The rat's lifecycle: how it is loaded, spawned into a group, joined to its nest's shared activity budget, serialised for the network, turned into a corpse, and handed back and forth between being a creature and being an item. |
| [`ai_rat.h`](ai_rat.h.md) | Declares the rat: the one creature in the game that is not built on the shared monster base, that is edible, and whose behaviour is a stack of states rather than a tree. |
| [`ai_rat_animations.cpp`](ai_rat_animations.cpp.md) | The rat's clip table and the rule that picks a clip from nothing but speed, turn angle, and whether the rat is biting — the animation layer for a creature with no action table. |
| [`ai_rat_behaviour.cpp`](ai_rat_behaviour.cpp.md) | The rat's think step — three calls — and the rule that makes a nest drift across the world graph over minutes rather than sitting on its spawn point forever. |
| [`ai_rat_feel.cpp`](ai_rat_feel.cpp.md) | What a rat is allowed to see, and what hearing something does to its nerve. |
| [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) | The bite, the flinch, the rule for judging a corpse worth eating, and the morale clock that decides whether a rat fights or runs. |
| [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md) | The rat's previous brain: a switch-driven loop that ran until a state declared itself settled. It is excluded from the build, it no longer compiles, and it is superseded by the state stack — but it is the only readable record of what several of the live predicates were originally for. |
| [`ai_rat_impl.h`](ai_rat_impl.h.md) | The nest's budget: how many rats in a group may be awake at once, how many may stand still, and the alarm that spreads a death to all of them. |
| [`ai_rat_inline.h`](ai_rat_inline.h.md) | The wandering loop in four short routines: when to pick a new goal, where that goal is, how fast to go, and how the countdown is clamped against a long frame. |
| [`ai_rat_space.h`](ai_rat_space.h.md) | The rat's five sounds and the masks that decide which of them may interrupt which. |
| [`ai_rat_templates.cpp`](ai_rat_templates.cpp.md) | The rat's own locomotion: it integrates a heading and a position every frame, asks the navigation mesh whether the step it just computed is legal, and backs the whole step out when it is not. This is what the rest of the game's creatures delegate to the path builder, and the rat does not. |
| [`rat_state_activation.cpp`](rat_state_activation.cpp.md) | What each rat state actually does on a tick once it has decided to stay: wander, settle, charge, bite, bolt, go home, or eat. |
| [`rat_state_initialize.cpp`](rat_state_initialize.cpp.md) | The three rat states that need something set up at the moment they are entered: where to flee to, where the startle came from, and whether to re-roll a wander goal. |
| [`rat_state_switch.cpp`](rat_state_switch.cpp.md) | The rat's vocabulary of questions and small actions: every condition a state may branch on, and every one-line thing a state may do, each named and exposed so that the states themselves are almost free of logic. |

**Not in this directory but part of the same subsystem**: the twelve state classes, the state interface and the state-stack manager live one level up among the `rat_state*` and `rat_states*` files of chapter 23. All twelve transition tables are there, not here.
