# src/xrEngine/pure.h

> The engine's broadcast mechanism — nine named per-frame/per-lifecycle signals, and the priority-ordered subscriber list they are delivered through.

**Needs** — [`xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md)
**Used by** — [`CustomHUD.h`](CustomHUD.h.md) · [`Device_destroy.cpp`](Device_destroy.cpp.md) · [`Engine.h`](Engine.h.md) · [`FDemoRecord.h`](FDemoRecord.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`StatGraph.cpp`](StatGraph.cpp.md) · [`StatGraph.h`](StatGraph.h.md) · [`Stats.h`](Stats.h.md) · [`XR_IOConsole.h`](XR_IOConsole.h.md) · [`device.cpp`](device.cpp.md) · [`device.h`](device.h.md) · [`editor_base.h`](editor_base.h.md) · [`pure.cpp`](pure.cpp.md) · _and 5 more_
**Tier floor** — T2: a list of subscribers sorted by priority, mutated while it is being walked. Nothing here needs manual memory or explicit layout; it is T1 in the original only because the whole module is.

## Purpose

The device drives the frame, but most of what happens in a frame is owned by modules the
device must not know about — the level, the UI, the sound world, the weather, the
in-game editor. This file defines the only coupling the device is allowed: a fixed set of
*signals*, and one registry type that delivers a signal to everyone who asked for it, in a
declared order.

The signal set is closed and small, and that is the point. A module cannot invent a new
frame phase; it can only choose which of the nine existing phases it participates in and
at what priority. The device's frame loop is therefore readable as a sequence of signal
broadcasts.

## State

```text
RECORD Subscriber<Signal>
  object   : handle to a receiver of Signal
  priority : int              # higher is delivered earlier

RECORD MessageRegistry<Signal>
  subscribers : list<Subscriber<Signal>>   # invariant: sorted descending by priority
                                           #            whenever not mid-broadcast
  changed     : bool          # a mutation arrived during a broadcast
  in_process  : bool          # a broadcast is on the stack
```

Invariants:

- No object appears twice in a registry with a live priority. Registering a second time is
  a programming error, caught in debug builds by a linear scan of the list.
- Unsubscribing never removes an entry during a broadcast; it *tombstones* the entry by
  setting its priority to the invalid sentinel. The compaction happens at the next resort.
  This is what makes it safe for a receiver to unsubscribe itself, or another receiver,
  from inside its own callback.
- After a resort, no tombstone remains: tombstones sort to the end and are popped.

## The signal set

Nine signals, each a one-method interface an implementor satisfies:

| Signal | Meaning |
|---|---|
| `Frame` | the per-frame update phase (the name is historical; it is the frame *start*) |
| `FrameEnd` | after rendering has been submitted for the frame |
| `Render` | contribute geometry to this frame's scene |
| `AppActivate` / `AppDeactivate` | the window gained or lost focus |
| `AppStart` / `AppEnd` | the application's outermost bracket |
| `DeviceReset` | the graphics device was recreated; every device-owned resource is gone |
| `UIReset` | the interface must rebuild itself — resolution or aspect changed |

`DeviceReset` and `UIReset` are separate because a device loss and a mere window resize
have different costs: the first invalidates GPU resources, the second only invalidates
layout.

## Priorities

Four named levels plus a sentinel:

```text
LOW      # runs last
NORMAL   # the default
HIGH     # runs first among ordinary subscribers
CAPTURE  # exclusive: see below
INVALID  # tombstone; never delivered
```

**Capture is the load-bearing one.** If the highest-priority subscriber holds the capture
priority, the broadcast delivers to *that subscriber only* and stops. This is how a modal
consumer — a console that has the keyboard, a video that owns the frame — takes a signal
away from everything else without any other module knowing it exists. There is no
"handled" return value anywhere in this system; exclusivity is expressed by priority
alone.

## `MessageRegistry`

**Contract** — a registry per signal type. `add(object, priority)` subscribes;
`remove(object)` unsubscribes (idempotent, and safe from inside a broadcast);
`process()` delivers the signal; `clear()` empties it. All of it is single-threaded: the
registries are touched only from the frame thread.

**Invariants** — see State. The one that matters to a rebuilder: mutation during a
broadcast must not reorder or reallocate the sequence being walked. Appending and
tombstoning both satisfy that; deleting does not.

```text
FUNCTION process(registry)
  IF registry.subscribers is empty THEN RETURN
  registry.in_process = true

  IF registry.subscribers[0].priority == CAPTURE THEN
    deliver signal to registry.subscribers[0].object    # exclusive: nobody else runs
  ELSE
    FOR EACH s IN registry.subscribers                  # by index: the list may grow
      IF s.priority != INVALID THEN deliver signal to s.object

  IF registry.changed THEN resort(registry)
  registry.in_process = false

FUNCTION resort(registry)
  sort registry.subscribers by priority, descending     # stability is not relied upon
  WHILE last entry has priority INVALID
    drop it
  registry.changed = false
```

**Notes** — the walk is by index rather than by cursor because a subscriber may subscribe
another during delivery; newly added entries land at the end and are visited in the same
broadcast. That is accepted behaviour, not a bug: a module that spawns a participant
mid-frame wants it to see the current frame.

The sort is a full re-sort on every mutation outside a broadcast. The lists are tens of
entries, so this is cheaper than maintaining order incrementally — but a rebuild that
finds itself with thousands of subscribers should insert in place instead.
