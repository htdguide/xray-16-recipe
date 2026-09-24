# src/xrEngine/pure_relcase.cpp

> The base that makes any non-object holder of object references get told before those references go stale.

**Needs** — [`pure_relcase.h`](pure_relcase.h.md) · [`xr_object_list.h`](xr_object_list.h.md) · [`IGame_Level.h`](IGame_Level.h.md)
**Used by** — [`pure_relcase.h`](pure_relcase.h.md)
**Tier floor** — T2: a registration list and a callback. Nothing device-facing.

## Purpose

When an object is destroyed, every *other* object is told, so it can drop its references —
the engine calls this a *relcase* (release-case) notification, and the object registry
broadcasts it to every registered object as part of the destruction pass. But plenty of
things that hold object references are not objects: camera effectors, HUD widgets, sound
players, AI helper caches. This file gives them the same notification without making them
objects.

The whole mechanism is two calls — register on construction, unregister on destruction —
and the only interesting part is *how the registration survives being removed from the
middle of a list*.

## State

```text
RECORD RelcaseSubscriber
  slot : int      # index into the level's relcase callback table
```

**Invariant** — the value in `slot` always equals this subscriber's current index in the
registry's table. The registry maintains this by writing back into the subscriber whenever
it moves an entry. That write-back is the entire trick: unregistration is O(1) because the
subscriber knows exactly where it lives, and the registry keeps that knowledge true.

## `RelcaseSubscriber`

**Contract** — constructed with a callback taking the object about to be destroyed.
Registration requires a level to exist and fails loudly if it does not, because there is
nowhere else to put the callback. Destruction unregisters, and tolerates the level having
already been torn down (during shutdown the whole table is gone, and the callback list
disappeared with it).

```text
FUNCTION RelcaseSubscriber.construct(callback)
  REQUIRE a level is loaded            # no level, no registry, no subscription
  level.objects.relcase_register(callback, address of this.slot)

FUNCTION RelcaseSubscriber.destruct()
  IF a level is loaded THEN
    level.objects.relcase_unregister(address of this.slot)
  # else: the level tore the whole table down already
```

**Notes** — in the original the callback is bound by taking the most-derived type's own
method and the subscriber's own address, which works because the subscriber is inherited
from. A rebuild should pass a closure; the inheritance is a way of stapling a callback to
an object, not a decision about type hierarchy.

The asymmetry — construction *requires* a level, destruction merely *checks* — is the
lifetime fact worth keeping: subscribers are created during level load and may outlive
level unload.
