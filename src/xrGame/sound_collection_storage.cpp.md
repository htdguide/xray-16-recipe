# src/xrGame/sound_collection_storage.cpp

> Interns sound collections by the value of their parameters, so that a hundred creatures with the same voice hold one copy of its samples.

**Needs** — [`sound_collection_storage.h`](sound_collection_storage.h.md) · [`sound_player.h`](sound_player.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`sound_collection_storage.h`](sound_collection_storage.h.md)
**Tier floor** — T2: owns device-backed audio handles for the process lifetime

## Purpose

Building a sound collection means probing the virtual filesystem for every numbered
variant of every name stem and loading each one (see
[`sound_player.cpp`](sound_player.cpp.md)). A level may hold dozens of creatures of the
same species, each registering the same dozen sound kinds. Without interning, that is the
same probe-and-load repeated dozens of times and the same samples resident dozens of
times. This file is the one line of defence against that, and it is why the parameter
record defines equality field by field.

The policy is deliberately simpler than the smart-cover description cache: **nothing is
ever evicted.** A collection loaded for one creature stays loaded until the storage itself
is destroyed at shutdown. That trades memory for the guarantee that a creature spawning
mid-level never stalls the frame on an audio load.

## State

```text
RECORD sound_collection_storage
  objects : list<(params, collection)>
  # invariant: params are unique across the list — the lookup depends on it
  # invariant: a returned entry's address is stable for the process lifetime,
  #            because callers store the collection and use it for years of play
```

## `object`

**Contract** — takes a collection parameter value; returns the existing entry with equal
parameters, or builds, stores and returns a new one. Never fails and never returns
nothing: a parameter set that matches no files on disk yields an empty collection, which
is legal. Allocates and performs filesystem probes plus audio loads on a miss, so a miss
can cost milliseconds; hits are a linear scan. Not thread-safe; called from the simulation
thread.

```text
FUNCTION object(params) -> (params, collection)
  FOR EACH entry IN objects
    IF entry.params EQUALS params        # field-wise over all four fields
      RETURN entry
  APPEND (params, build_collection(params)) TO objects
  RETURN last entry
```

**Invariants** — the returned reference must stay valid after further insertions, because
callers keep it. In the original this is guaranteed by the container's growth behaviour
being irrelevant — only the collection *behind* the entry is retained. A rebuild should
return the collection by a stable handle rather than by a reference into the table.

**Notes** — the linear scan is not a mistake. The table holds on the order of tens of
distinct parameter sets for a whole level, and lookups happen at creature registration,
not per frame; a hash on a four-field key would cost more to maintain than it saves.

## `~sound_collection_storage` (teardown)

**Contract** — destroys every collection, which releases every loaded sample to the audio
device. Must run while the audio device is still alive, which places it before device
shutdown in the process teardown order. No assertion guards that; a rebuild should add
one, since releasing audio handles after the device is gone is the classic failure of this
ordering.
