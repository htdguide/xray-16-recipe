# src/xrEngine/Feel_Touch.cpp

> Maintains "which entities are inside my radius right now", with enter and leave edges and a temporary exclusion list.

**Needs** — [`Feel_Touch.h`](Feel_Touch.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`xr_object.h`](xr_object.h.md) · [`device.h`](device.h.md)
**Used by** — reached through its declarations in [`Feel_Touch.h`](Feel_Touch.h.md); callers name that, not this file.
**Tier floor** — T2: a spatial query and two set differences.

## Purpose

Anomalies, trigger zones, fire, radiation, and anything else that acts on whatever is near it all need the same thing: a set that tracks membership over time and reports the transitions, not just the current contents. Computing the set is a spatial query; the value this adds is the *edges* and the exclusion list.

## State

```text
RECORD Touch
  contacts    : list<entity>              # the current set; publicly readable
  scratch     : list<entity>              # this update's query result; reused
  denied      : list<(entity, expiry_ms)> # temporarily excluded

  # invariant: an entity appears in contacts at most once
  # invariant: an entity in denied never enters contacts while its exclusion holds
```

## `update`

**Contract** — Given a centre and a radius, recomputes the contact set and fires an enter notification for each newly contacting entity and a leave for each that has stopped. Expires stale exclusions first. Entities marked for destruction are excluded from both directions — they are never entered, and if already in the set they are left. Does not allocate in the steady state: the scratch buffer is reserved to the current set's size and reused.

```text
FUNCTION update(centre, radius)
  now = global_time_ms
  remove every entry from denied whose expiry has passed

  scratch = spatial query: every entity within radius of centre

  # Enter edges
  FOR EACH candidate IN scratch
    IF candidate is marked for destruction THEN CONTINUE
    IF NOT contact(candidate) THEN CONTINUE          # the game's own filter
    IF candidate is already in contacts THEN CONTINUE
    IF candidate is in denied THEN CONTINUE
    contacts.append(candidate) ; on_enter(candidate)

  # Leave edges
  FOR EACH held IN contacts
    IF held is marked for destruction
       OR NOT contact(held)
       OR held is not in scratch
    THEN remove it from contacts ; on_leave(held)
```

**Invariants** — A leave notification is fired for every entity that ever received an enter, including on destruction. Game code relies on that pairing to undo whatever the enter did — stop a damage effect, release a sound.

**Notes** — The game's per-candidate filter is consulted on *both* passes, not just on entry. That is what lets a filter whose answer changes over time — "is this entity still alive", "is it still on the ground" — push an entity out of the set without it having moved.

**Notes** — The spatial query returns every entity in range; the radius is not re-tested here. The query is the radius test, and the filter is the only other condition.

**Notes** — The exclusion list uses the *global* millisecond clock, which does not pause. An exclusion therefore expires while the game is paused. For the durations involved — the exclusion is used to stop an entity being re-grabbed immediately after being thrown out — that is harmless, but it is a different clock from the one the rest of the sense system uses.

**Notes** — The original contains a commented-out call, at the end of the update, that would yield to the frame scheduler. Its removal is why a touch update is atomic: a scheduler slice in the middle of a set difference would let entities be destroyed between the two passes.

## `contact`

**Contract** — The per-candidate filter. The default accepts every candidate; the game overrides it to test distance-to-shape, entity class, or any other predicate.

## `deny`

**Contract** — Excludes an entity from entering the set for a duration in milliseconds. It is not removed if it is already in the set — the exclusion only blocks re-entry.

## `on_object_released`

**Contract** — Called when an entity anywhere in the process is destroyed. Removes it from the contact set, firing its leave notification, and from the exclusion list. This is the engine-wide invariant that a destroyed entity is unreferenced everywhere before its memory is released, and it is why this type participates in the release-case chain at all.

**Notes** — It fires the leave notification, unlike the ordinary removal path which would have fired it on the next update. Firing it here is what guarantees the enter/leave pairing holds even for an entity that vanishes.
