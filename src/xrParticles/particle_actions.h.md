# src/xrParticles/particle_actions.h

> The action interface every simulation step is built from, and the ordered, lockable list
> that holds a sequence of them.

**Needs** — [`psystem.h`](psystem.h.md) · [`xrCore/Threading/Lock.hpp`](../xrCore/Threading/Lock.hpp.md)
**Used by** — [`ParticleEffectDef.cpp`](../Layers/xrRender/ParticleEffectDef.cpp.md) · [`particle_actions.cpp`](particle_actions.cpp.md) · [`particle_actions_collection.h`](particle_actions_collection.h.md) · [`particle_manager.h`](particle_manager.h.md)
**Tier floor** — T2: an interface plus a guarded ordered collection. Nothing here touches a
byte layout; the T1 floor of the chapter comes from its neighbours.

## Purpose

This is the chapter's central abstraction, and it is deliberately small: an *action* is a
thing that can be applied to a whole pool of particles for a duration, that can be told where
its emitter now is, and that can be read from and written to a file. That is the entire
contract. Everything an effect does — emitting, moving, colouring, killing — is one of
thirty-one implementations of it, enumerated in
[`particle_actions_collection.h`](particle_actions_collection.h.md).

A file is therefore not a program with branches; it is a *straight sequence* of these, applied
in order, every step, to every live particle. No action can skip another, none can loop, and
the retired "call another action list" code (slot 2 of the action enumeration) is the vestige
of the one attempt to allow it.

## State

```text
INTERFACE Action
  flags : int (32-bit)      # bit 1 = this action's parameters rotate with the emitter;
                            # cleared, only the emitter's translation applies to them.
                            # Bits 29-31 are claimed by the source action for its own use.
  kind  : ActionKind

  execute(pool, dt, lifetime_hint)   # advance every live particle by dt
  transform(placement)               # re-derive world-space parameters from local ones
  load(reader)                       # read parameters from an authored stream
  save(writer)
```

```text
RECORD ActionList
  actions : list<Action>    # ordered; order is the program
  locked  : bool            # true while a walk over the list is in progress
```

Invariants:

- The list owns its actions; clearing or destroying the list destroys them.
- The list may not be mutated — cleared, appended to, resized — while a walk is in progress.
  Both conditions are checked, and a violation is a hard failure rather than a tolerated race.
- Mutation and walking are both serialized by the list's own lock, so an update running on a
  worker thread and a load running on the main thread cannot interleave.

**Notes** — the walk-in-progress flag is redundant with the lock in the sense that the lock
already excludes concurrent walkers; it is not redundant against *re-entrancy* — an action
that reached back into its own list while executing would deadlock or corrupt, and this turns
that into a diagnosable failure. A rebuild keeps the check and can drop the flag if its
concurrency primitive already detects re-entrant acquisition.

## `Action`

**Contract** — as above. Four operations, no state visible to the caller beyond the flag word
and the kind.

`execute` receives the pool, the step duration in seconds, and a mutable `lifetime_hint`. That
third parameter is a one-slot channel *between actions in the same list*: the manager sets it
to 1 before the walk, the kill-old action overwrites it with its age limit, and the
target-colour action reads it to convert its fractional time window into seconds. This makes
the list order semantically load-bearing in a way nothing in the file format records — see
[`particle_actions_collection.cpp`](particle_actions_collection.cpp.md). A rebuild that wants
the channel to be explicit should make the step carry a small context record rather than a
single mutable scalar.

`transform` is not called per step. It is called whenever the emitter's placement changes, and
it recomputes each action's world-space parameters from the authored local-space ones. Every
action that has spatial parameters keeps *both* copies.

## `ActionList`

**Contract** — an ordered, owning, lockable sequence with append, clear, resize, size, and
iteration. Nothing addresses an action by index from outside; the list is walked, never
indexed.

**Notes** — the list reserves room for four actions on creation. Typical shipped effects hold
between three and eight actions, so that reservation avoids a reallocation for the common
case; it carries no other meaning.
