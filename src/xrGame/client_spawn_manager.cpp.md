# src/xrGame/client_spawn_manager.cpp

> Lets one object say "tell me when the object with this identifier comes into existence", and delivers the notification exactly once.

**Needs** — [`client_spawn_manager.h`](client_spawn_manager.h.md) · [`client_spawn_manager_inline.h`](client_spawn_manager_inline.h.md) · [`Level.h`](Level.h.md) · [`GameObject.h`](GameObject.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: two levels of map and a one-shot dispatch

## Purpose

An entity identifier can be known long before the object it names exists as a client object.
A saved game restores a stalker who remembers the identifier of the weapon he was carrying;
a script is told the identifier of an entity that is still offline; an object is spawned
before the object that will own it. Anything that wants to act on such an identifier has to
wait, and polling every frame for an object that may never appear is both wasteful and easy
to leak.

This registry is the alternative. A waiter registers a callback against an identifier; when
an object with that identifier is created, every callback waiting on it fires and the entry
is discarded. If the object already exists, the callback fires immediately and nothing is
stored — which means a caller never has to check first, and that is the whole ergonomic
point.

## State

```text
RECORD SpawnCallback
  native_callback : optional<function(object)>      # engine-side waiter
  script_callback : optional<script function>       # script-side waiter, with an optional
                                                    # bound receiver object

registry : map<awaited_id, map<waiter_id, SpawnCallback>>
```

**Invariants**

- The outer key is the identifier being **waited for**; the inner key identifies **who is
  waiting**. That pairing is what allows a waiter to be removed without disturbing anyone
  else waiting on the same object, and allows everything a waiter is waiting on to be torn
  down when the waiter dies.
- At most one callback per (awaited, waiter) pair. Registering a second for the same pair
  **replaces** the first, both halves of it, rather than chaining. A waiter that wants two
  things done on one spawn must do them in one callback.
- Delivery is one-shot: firing an entry clears the whole awaited-identifier bucket, including
  any callback registered during delivery. See the note below — that is a real trap.
- An empty inner map is removed rather than left behind, so the registry's size is the number
  of identifiers actually being awaited, which is what makes the leak check at destruction
  meaningful.
- **The registry must be empty when the manager is destroyed.** A non-empty registry at
  shutdown means some waiter registered for an object that never spawned and never
  deregistered — usually an object destroyed without cleaning up — and the assertion is the
  only thing that catches it.

## `add`

**Contract** — registers a callback to fire when the awaited identifier's object exists.
Four entry points differing only in what the callback is: a script function, a script
function with a bound receiver, an engine-side function, and a prepared callback record.
All funnel into one path.

```text
FUNCTION add(awaited_id, waiter_id, callback)
  object = find live object with awaited_id
  IF object exists THEN
    deliver(callback, object)      # fire now; store nothing
    RETURN

  IF registry has no bucket for awaited_id THEN
    registry[awaited_id] = { waiter_id: callback }
  ELSE IF that bucket has no entry for waiter_id THEN
    registry[awaited_id][waiter_id] = callback
  ELSE
    replace both halves of the existing entry with this one
```

**Invariants** — the immediate-delivery branch is the contract, not an optimization. Callers
rely on "register and it will happen", and if the object is already there the only way to
honour that is to call synchronously — which means **a caller must be prepared for its
callback to run before `add` returns**.

**Notes** — the operation that replaces an existing entry is named as if it merged the two.
It does not: it overwrites both the native and the script half. A rebuild should name it for
what it does, and should consider whether replacing is right — a second registration silently
cancelling a first is a plausible source of quiet bugs in mod scripts.

## delivery

**Contract** — invokes both halves of a callback record: the engine-side function with the
object, then the script function with the object's identifier **and** its script-visible
facade. Both are passed because a script waiting on a spawn usually wants the facade, while
the identifier is what it registered with and what it can correlate.

```text
FUNCTION deliver(callback, object)
  IF callback.native_callback exists THEN callback.native_callback(object)
  facade = object's script facade, or none if it has none
  callback.script_callback(object.id, facade)
```

**Notes** — an object with no script facade yields a null facade rather than a skipped call.
Scripts must tolerate it; in practice everything a script waits for has one.

## firing on spawn

**Contract** — called with each newly created client object. Fires every callback waiting on
that object's identifier, in registry order, then discards the whole bucket. Does nothing if
nobody is waiting, which is the common case and must therefore be one lookup.

```text
FUNCTION on_object_spawned(object)
  bucket = registry[object.id]
  IF bucket is absent THEN RETURN
  FOR EACH (waiter_id, callback) IN bucket
    deliver(callback, object)
  remove registry[object.id]
```

**Invariants** — the bucket is cleared *after* all deliveries, so a callback that registers a
new wait on the same identifier during delivery has its registration thrown away. That is a
trap with no diagnostic; a rebuild should either take the bucket out of the registry before
iterating it, or reject re-registration during delivery.

## `remove`

**Contract** — deregisters one waiter from one awaited identifier, dropping the bucket if it
becomes empty. A pair that is not registered is a script error and is logged as one, because
the usual cause is a script deregistering twice or deregistering from the wrong object, and
silence there costs hours.

## `clear(waiter_id)`

**Contract** — removes this waiter from every awaited identifier's bucket. Called when an
object is destroyed: it may be waiting on several things and must leave none of them behind,
or the destructor's emptiness check fires at shutdown with no clue which object was at fault.

**Notes** — this walks every bucket and attempts a removal from each, so for every identifier
the waiter was *not* waiting on it takes the "not registered" path — **and logs a script
error**. The intent was clearly to suppress that: the call passes a suppression flag, and the
removal declares one. The removal never reads it. The result is a burst of spurious errors on
every object destruction, and a rebuild must honour the flag.

## the reverse lookup

**Contract** — returns the callback registered for a pair, or nothing.

**Notes** — **it indexes the registry the other way round from every other operation**: it
treats its first argument as the waiter and its second as the awaited identifier, where
`add` and `remove` do the opposite. Either the accessor or its two callers are wrong; the
source does not say which. A rebuild should settle on one argument order and use it
everywhere.

## `clear()` and the debug dumps

**Contract** — `clear()` drops the whole registry, used when a level is unloaded and every
client object goes away at once. The two debug dumps print what is still being awaited and by
whom; they exist to diagnose exactly the leak the destructor asserts on.

**Notes** — the dumps print their identifiers with the awaited and waiting roles transposed in
their message text. Cosmetic, but misleading in precisely the situation they are read in.
