# src/xrGame/GlobalFeelTouch.cpp

> A touch-sense participant that senses nothing and exists only to hold a set of temporarily ignored objects with expiry times.

**Needs** — [`GlobalFeelTouch.hpp`](GlobalFeelTouch.hpp.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — reached through its declarations in [`GlobalFeelTouch.hpp`](GlobalFeelTouch.hpp.md); callers name that, not this file.
**Tier floor** — T3: a list with time-based expiry

## Purpose

The touch sense (see the glossary: *feel*) lets an object learn which objects are near it,
and lets it *deny* specific objects for a while — a thing you just dropped should not
immediately be picked up again, a grenade you just threw should not collide with you.
The deny list, with its per-entry expiry, is a feature of the base sense. This subclass
uses that feature and nothing else: it never asks who is nearby.

It exists as a process-wide singleton so that a deny decision made by one part of the
game is visible to another that does not own a sense of its own.

## State

The deny list is inherited from the touch sense and is a list of (object, expiry time)
pairs. Invariant: an entry whose expiry has passed is treated as absent, and is pruned on
the next update — so a lookup that runs between expiry and pruning could still see a
stale entry. The lookup here does *not* prune first (the code that would have done so is
disabled), so denial can outlive its expiry by up to one update interval.

## `feel_touch_update`

**Contract** — the sense's per-update hook, which normally receives the query position
and radius and recomputes the nearby set. This implementation ignores both arguments and
only prunes the deny list: every entry whose expiry is at or before the current global
time is removed. Allocates nothing; the list is compacted in place.

## `is_object_denied`

**Contract** — answers whether a given object is currently on the deny list. Linear scan;
the list is expected to hold a handful of entries at most. Does not prune, so see the
staleness note above.

```text
FUNCTION is_object_denied(object) -> bool
  RETURN any entry IN deny_list HAS entry.object == object
```

**Notes** — comparison is by object identity, not by entity identifier, which matters
because the list may briefly hold an object that has been destroyed. Callers are
responsible for not denying an object that will outlive them; nothing here validates it.
