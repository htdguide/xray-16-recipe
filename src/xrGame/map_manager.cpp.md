# src/xrGame/map_manager.cpp

> Owns every map marker: creates them, finds them, advances them on a staggered schedule, and reaps the ones whose subject is gone.

**Needs** — [`map_manager.h`](map_manager.h.md) · [`map_location.h`](map_location.h.md) · [`map_location_defs.h`](map_location_defs.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`Level.h`](Level.h.md) · [`Actor.h`](Actor.h.md) · [`InventoryOwner.h`](InventoryOwner.h.md) · [`relation_registry.h`](relation_registry.h.md) · [`GametaskManager.h`](GametaskManager.h.md) · [`game_object_space.h`](game_object_space.h.md)
**Used by** — [`map_manager.h`](map_manager.h.md)
**Tier floor** — T3: a registry with a per-frame sweep

## Purpose

Markers are created by scripts, by tasks, by the relation system and by the engine, and they
outlive the things that made them. This file is the single owner: one list per level, stored
inside the save registry so that persistence is automatic, plus the lifetime discipline that
keeps a marker from being destroyed while something is still drawing it.

## State

```text
RECORD MapManager
  locations   : list<LocationKey>       # borrowed from the per-level save registry
  destroy_queue : list<MapLocation>     # markers to free at the START of the NEXT frame
  spot_descriptions : parsed document   # shared; every marker reads its widgets from it
```

**Invariant** — the marker list is **not owned here**. It lives in the save registry, which is
what makes markers persist across a save without the manager serializing anything itself. The
manager holds a borrowed reference, resolved lazily on first use and dropped when the registry
is replaced by a load.

**Invariant** — a removed marker is **never freed immediately**. It goes on a queue that is
drained at the beginning of the next frame's update. Markers are referenced by the map
widgets, by the task manager and by iteration in progress; freeing one mid-frame leaves a
dangling reference in whichever of those is currently holding it. A rebuild without manual
lifetimes still needs the deferral, because the *logical* question — is this marker still
valid — is what the other systems are asking.

**Invariant** — the list is kept sorted so that **stale markers occupy the tail**. Reaping is
then a truncation, which is what makes it safe to remove many at once.

## `Update`

**Contract** — the per-frame drive, in a fixed order: drain last frame's destruction queue,
advance every marker, re-sort, then reap from the tail.

```text
FUNCTION update()
  free everything queued last frame

  FOR EACH marker AT index i
    marker.actual = marker.update()                  # refreshes its own frame cache
    IF marker.actual AND (frame MOD 3) == (i MOD 3) THEN
      marker.recalculate_position()                  # the expensive part, one third per frame

  sort so that stale markers move to the tail
  WHILE the last marker is stale
    tell the task manager it is going away
    queue it for destruction ; drop it from the list
```

**Invariant** — the position recalculation is **staggered across three frames**: each marker's
index decides which of every three frames it recomputes on. Position is the costly part — it
may mean an alife lookup and a graph query — and a map with two hundred markers cannot afford
all of them every frame. The visible consequence is that a marker's position can be up to two
frames stale, which at map-drawing scale is invisible.

Three is the shipped divisor with no derivation. Note the stagger is by *list index*, which
changes when the list is re-sorted, so a given marker does not reliably update on the same
phase — the effect is a roughly even spread rather than a strict rotation, which is all it
needs to be.

**Invariant** — the task manager is notified before a marker is destroyed, on every removal
path in this file without exception. A task holding a marker that has been freed will
dereference it the next time it draws.

**Notes** — the destruction queue is drained at the *start* of the update rather than the end,
which guarantees a full frame has elapsed since the marker was removed. Draining at the end
would free a marker within the same frame it was removed.

## `AddMapLocation`

**Contract** — creates a marker of a named spot type for an entity, appends it to the list and
returns it. **Does not check for a duplicate**: two calls with the same spot type and entity
produce two markers, both of which draw. That is deliberate — several tasks can mark the same
character — and it is why every lookup here comes in a first-match and an all-matches form.

In single player the actor is notified through a script callback that a marker was added,
which is how a mod can react to the engine's own marker creation.

## `AddRelationLocation`

**Contract** — creates the relation marker for one character, with the spot type chosen from
the current standing between that character and the player — or the corpse spot type if he is
dead. Requires a current view entity; returns nothing without one.

**Unlike the plain add, this one deduplicates**: if a marker of the chosen spot type already
exists for that character, its time to live is refreshed and the existing marker is returned.
Relation markers are created every time a character is noticed, which is constantly, and
without the deduplication one character would accumulate a marker per sighting.

**Notes** — the deduplication searches by *spot type*, so a character whose standing has
changed since his marker was made is not found and a second marker is created. The first one
then re-derives its own spot type on its next update and the two collide. This is why the
relation marker suppresses duplicate presentations of itself — see
[`map_location.cpp`](map_location.cpp.md). A rebuild should search by entity and marker kind
instead, and the suppression becomes unnecessary.

A development build reports when the deduplication finds a marker that is *not* a relation
marker, because that means a script has claimed the relation spot type for its own use.

## `RemoveMapLocation` · `RemoveMapLocationByObjectID` · `RemoveMapLocation` (by handle)

**Contract** — three removals for three ways of naming a marker: by (spot type, entity), by
entity alone — which removes *every* marker on that entity, looping until none remain — and by
handle. All three notify the task manager and queue the marker for destruction rather than
freeing it.

**Notes** — the by-entity form re-searches from the beginning of the list after each removal
rather than continuing from where it was. Correct, quadratic in the number of matches, and
irrelevant at the scale involved.

## `OnObjectDestroyNotify`

**Contract** — the notification an entity sends as it is destroyed: remove every marker on it.
Without this, a marker survives its subject and reports itself non-actual on the next update,
which is one frame of a marker pointing at nothing.

## `GetMapLocation` · `GetMapLocations` · `HasMapLocation` · `GetMapLocationsForObject`

**Contract** — the four lookups: the first marker matching a (spot type, entity) pair; all
markers matching it, appended to a caller's list; whether any does; and every *actual* marker
on an entity, replacing a caller's list.

**Invariant** — the last one filters to actual markers and the others do not. It is the one
used by the relation marker's duplicate suppression, which must not be confused by a marker
that is already on its way out.

All four are linear scans. The list is a few hundred entries and they run per frame; that is
the shipped cost.

## `Locations`

**Contract** — resolves and caches the borrowed reference to the save registry's marker list
on first use. `ResetStorage` clears it, which a save load must do — loading replaces the
registry wholesale, and the cached reference would point into the discarded one.

**Invariant** — the registry is initialized for a single slot at construction. Markers are
per-level, and the registry's own per-level keying is what separates them; the manager sees
only the current level's list.

## `OnUIReset`

**Contract** — reloads the map-spot description document and asks **every existing marker** to
rebuild its widgets from it. Called when the user interface is reset — a resolution change, a
skin change. This is the reason `LoadSpot` had to be safe to call on a marker that already has
widgets.

## `DisableAllPointers`

**Contract** — clears the pointer on every marker. Used when the map screen changes what it is
tracking: pointers are re-enabled selectively afterwards rather than toggled individually.

## `SLocationKey::save` · `SLocationKey::load`

**Contract** — the per-marker save format: entity identifier, spot type, **one reserved
byte**, then the marker's own payload. Loading reconstructs the marker from the spot type and
the entity before letting it read its payload, which is why the spot type is written first.

**Notes** — the reserved byte is written as zero and read and discarded. It is a field nobody
reads, retained because the shipped games' saves contain it. A rebuild must write and skip it
to stay compatible, and must not attach meaning to it.

## `CMapLocationRegistry::save`

**Contract** — writes the whole registry: the number of levels, then per level the level
identifier, the count of **serializable** markers, and those markers. The count requires a
first pass to compute, because the filter is per marker; a rebuild able to write a length
afterwards does it in one pass.

## Construction and teardown

**Contract** — construction parses the map-spot description document from the user-interface
configuration path (with a default path as fallback, so a partial skin still works) and
initializes the registry wrapper. Teardown drains the destruction queue, frees the registry
and releases the document.

**Invariant** — teardown drains the queue *first*. Markers in the queue are still referenced
from nothing by then, but freeing the registry first would free the list those markers came
from while they are still queued.
