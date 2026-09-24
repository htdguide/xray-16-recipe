# src/xrParticles/particle_manager.cpp

> The module's front door: it owns the pools and the action lists, runs one step by walking a
> list in order, and translates "play" and "stop" into pokes at individual actions.

**Needs** — [`particle_manager.h`](particle_manager.h.md) · [`particle_effect.h`](particle_effect.h.md) · [`particle_actions_collection.h`](particle_actions_collection.h.md) · [`psystem.h`](psystem.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: handle tables, a lock, and a dispatch over the action kinds.

## Purpose

Two jobs, and the second one is the interesting one.

The first is bookkeeping: pools and action lists live in two tables, handles are indices into
them, and a destroyed entry leaves a hole that the next creation reuses. That is all there is
to say about it.

The second is that the manager is where the *step* is defined — the order in which actions
run, the one value they pass between each other, and what "playing" and "stopped" mean for a
system that has no play state of its own. An effect here has no timeline, no phase and no
enabled bit. Starting one means finding its source action and clearing that action's silence
flag; stopping one means setting it again.

## State

```text
RECORD ParticleManager
  effects      : list<optional<ParticleEffect>>    # handle = index; holes are reused
  action_lists : list<optional<ActionList>>        # likewise
  lock         : mutex                             # guards both tables
```

Invariants:

- A handle is valid for the life of the process once issued, in the sense that it always
  indexes the table; whether the slot still holds anything is not checked by the lookups,
  which return the hole. Every caller assumes it does not destroy a handle it still uses.
- The tables only grow. A destroyed slot is cleared and re-filled by the next creation of the
  same kind, so handles are recycled and a stale handle can silently address a different
  effect. Callers hold their handles for the life of the object that owns them, which is why
  this has never bitten.
- The lock covers table lookup and mutation only. It is *not* held during a step; the action
  list's own lock covers that.

## `create_effect(max_particles)` / `destroy_effect(handle)`

**Contract** — allocates a pool of the given capacity, installs it in the first empty slot (or
appends), and returns that slot's index. Destroy releases the pool and empties the slot.
Both take the table lock. A handle outside the table is a hard failure, not a returned error.

## `create_action_list()` / `destroy_action_list(handle)`

**Contract** — as above, for an empty action list.

## `play(effect, list)`

**Contract** — starts an effect by reaching into three kinds of action and resetting their
private state. Takes the action list's lock for the whole walk. Touches no particles: a
played effect that already holds particles keeps them.

```text
FUNCTION play(effect, list)
  LOCK list DURING
    FOR EACH action IN list
      SWITCH action.kind
        Source     : action.silent = false      # start emitting
        Explosion  : action.age = 0             # restart the shock wave at the centre
        Turbulence : action.age = 0             # rewind the noise field's scroll
```

**Notes** — those three are exactly the actions that carry time of their own. Every other
action is a pure function of the particle and the step, so there is nothing in it to rewind.
That is a property of the action catalogue a rebuild must preserve: if a new action carries
internal time, it has to be added here too or an effect will not restart cleanly.

## `stop(effect, list, deferred)`

**Contract** — silences every source action in the list. When `deferred` is true — the normal
case — that is all: existing particles continue to run their actions and age out naturally, so
a torch that is switched off still lets its last embers rise and fade. When `deferred` is
false, the pool's live count is additionally zeroed, discarding every particle at once with no
death callbacks.

**Notes** — the hard stop's discard bypasses the pool's removal path entirely, so the death
callback never fires and any child effects bound to those particles are orphaned. The group
layer compensates by stopping its children itself. A rebuild that routes the hard stop through
per-particle removal will fire a storm of callbacks the original never fires.

## `update(effect, list, dt)`

**Contract** — one simulation step. Walks the list in order, applying every action to the whole
pool for `dt` seconds. Holds the action list's lock for the walk, so no action can be added or
removed mid-step. Allocates nothing. This is the only entry point that advances anything.

```text
FUNCTION update(effect, list, dt)
  pool = effects[effect] ; actions = action_lists[list]
  LOCK actions DURING
    lifetime_hint = 1.0            # see below
    FOR EACH action IN actions
      action.execute(pool, dt, lifetime_hint)
```

**Invariants** — the list is applied in authored order, once, with the same `dt` for every
action. There is no sub-stepping, no ordering by kind, and no parallelism across actions:
actions mutate the same pool and several of them (the kill actions, the source) change how
many particles exist, so the walk is inherently serial. Individual actions may parallelize
*within* themselves over the pool, and one of them explicitly does not because measurement
said the noise evaluation was faster serial.

**Notes** — `lifetime_hint` is the whole inter-action protocol, and it is worth spelling out
because nothing in an authored file records it. It starts each step at 1. The kill-old action
overwrites it with its configured age limit. The target-colour action multiplies its authored
`from`/`to` fractions by it to get an absolute age window. So:

- a list whose colour action precedes its kill-old action interprets that action's time window
  against a lifetime of one second, whatever the real lifetime is;
- a list with no kill-old action at all does the same.

Authored effects put kill-old first for this reason. A rebuild must apply the actions in order
and must carry this value forward, or colour ramps on long-lived particles collapse into the
first second of life.

Three actions each pass over the pool in *reverse* — the two kill actions and the caller's own
collision pass — because removal fills the hole from the end of the live prefix. See
[`particle_effect.cpp`](particle_effect.cpp.md).

## `transform(list, placement, parent_velocity)`

**Contract** — tells every action in the list where its emitter now is. Called when the
emitter moves, not per step. Each action recomputes its world-space parameters from its
authored local-space ones; actions with no spatial parameters do nothing. Additionally, every
source action's inherited velocity is set to the emitter's velocity scaled by that source's
own inheritance factor. Takes the list lock.

```text
FUNCTION transform(list, placement, parent_velocity)
  LOCK list DURING
    translation_only = placement with its rotation dropped
    FOR EACH action IN list
      # An action opts in to rotating with its emitter. One that does not still
      # follows it: a gravity direction authored as "down" stays down when the
      # emitter tips over, but an emission cone authored as "forward" turns with it.
      m = action.rotates_with_emitter ? placement : translation_only
      action.transform(m)
      IF action.kind = Source THEN
        action.inherited_velocity = parent_velocity · action.inheritance_factor
```

**Invariants** — the authored (local) parameters are never written by this call; only the
derived world copies are. That is why every spatial action carries two of everything, and it
is what lets an emitter be moved arbitrarily often without accumulating drift.

## `create_action(kind)`

**Contract** — constructs the action of the given kind with its fields uninitialized, stamps
the kind onto it and returns it. An unknown kind is a hard failure. The caller is expected to
fill the action by loading it; a created-but-unloaded action holds garbage.

**Notes** — two kind codes construct the same action as another code: the "derivative" variants
of target-rotate and target-velocity. See [`psystem.h`](psystem.h.md).

## `load_actions(list, reader)`

**Contract** — replaces the list's contents with the actions in the stream and returns how
many were read. An empty stream leaves the list empty rather than failing. Every action is
constructed from its kind code and then reads its own payload; a malformed stream is not
detectable and will desynchronize silently.

```text
FUNCTION load_actions(list, reader) -> int
  clear list                                  # destroys the previous actions
  IF reader is empty THEN RETURN 0
  count = reader.read_int32()
  REPEAT count TIMES
    action = create_action(ActionKind(reader.read_int32()))
    action.load(reader)                       # consumes the common header, then the payload
    append action to list
  RETURN list.size
```

**Notes** — the framing is described in full in
[`particle_actions_collection_io.cpp`](particle_actions_collection_io.cpp.md), including the
asymmetry between this reader and the writer below.

## `save_actions(list, writer)`

**Contract** — writes the action count followed by each action's own serialization. Takes the
list lock.

**Notes** — this path has no caller in the engine: authored effect files are produced by a
tool that is not part of this codebase, and the engine only ever reads them. It does not
round-trip with `load_actions` — see the io twin. A rebuild that needs a writer should derive
it from the reader rather than from this.
