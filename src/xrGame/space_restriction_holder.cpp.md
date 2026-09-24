# src/xrGame/space_restriction_holder.cpp

> The level's registry of restrictor volumes: it canonicalizes a comma-joined name list into a cache key, hands out one shared handle per distinct list, swaps geometry in and out as restrictor entities spawn and despawn, and maintains the two level-wide default restriction lists.

**Needs** — [`space_restriction_holder.h`](space_restriction_holder.h.md) · [`space_restriction_holder_inline.h`](space_restriction_holder_inline.h.md) · [`space_restrictor.h`](space_restrictor.h.md) · [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_shape.h`](space_restriction_shape.h.md) · [`space_restriction_composition.h`](space_restriction_composition.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — reached through its declarations in [`space_restriction_holder.h`](space_restriction_holder.h.md); callers name that, not this file.
**Tier floor** — T2: string canonicalization and a keyed registry with timed reclamation

## Purpose

Restriction lists arrive as text, from spawn records and from scripts, in whatever order the
author wrote them. Two entities restricted to the same two regions must share one border
computation. This file is where text becomes identity: it sorts the names within a list so
that any permutation yields the same key, and files one handle per key.

It is also where the two *default* lists live — the restrictions every entity on the level
inherits unless it says otherwise — and where restrictor spawn and despawn is turned into a
geometry swap that nobody holding a handle has to notice.

## State

```text
RECORD RestrictionHolder
  restrictions : map<text, RestrictionBridge>   # key: normalized name list
  default_out  : text                           # normalized; volumes everyone must stay inside
  default_in   : text                           # normalized; volumes everyone must stay out of

CONSTANT time_to_remove_garbage   = 300000      # 5 minutes of world clock
CONSTANT max_restrictions_per_list = 128
```

**Invariants**

- Every key is normalized. A lookup with an un-normalized list must normalize first or it
  will miss and build a duplicate.
- A single restrictor's own name is a valid key, and under it lives either a shape (the
  restrictor is spawned) or a single-name placeholder composition (it is not). The bridge
  object under that key survives both transitions, which is what makes handles stable.
- Shapes are never reclaimed. They belong to live restrictor entities; the registry only
  reclaims compositions.

## `normalize_string`

**Contract** — sort a comma-joined name list lexicographically and rejoin it. Returns the
input unchanged when it contains no comma, and the empty text for empty input. Hard-fails
above the per-list cap. Allocates only scratch.

```text
FUNCTION normalize(list) -> text
  IF list is empty          RETURN ""
  parts = split(list, ",")
  IF parts has one element  RETURN list       # nothing to reorder
  REQUIRE count(parts) < max_restrictions_per_list
  SORT parts lexicographically
  RETURN join(parts, ",")
```

**Invariants** — sorting is what makes the key canonical, and lexicographic order is chosen
only because it is total and cheap; nothing downstream depends on which order it is, only
that it is the same every time. The cap exists because the scratch storage is sized from
it; it is a generous bound on how many restrictors one entity is ever given, not a game
rule.

**Notes** — no deduplication happens here. A list naming the same restrictor twice produces
a key with it twice, a composition with two identical members, and a correct but wasteful
result. Callers that build lists — the join operation in
[`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) — deduplicate
themselves, which is where the responsibility actually sits.

## `restriction`

**Contract** — hand out the shared handle for a name list, creating it on first ask.
Returns nothing for an empty list, which is how "unrestricted" travels. Runs the garbage
collector on a miss.

```text
FUNCTION restriction(names) -> optional<handle>
  IF names is empty  RETURN none
  key = normalize(names)
  IF key IS IN restrictions  RETURN restrictions[key]
  collect_garbage()
  bridge = new bridge wrapping a new composition(self, key)
  restrictions[key] = bridge
  RETURN bridge
```

**Notes** — collecting only on a miss ties reclamation cost to the rate at which new
combinations appear, which is bursty at level load and near zero afterwards. It also means
a level where nothing new is ever asked for never reclaims, which is harmless: the objects
are small and the level ends.

## `register_restrictor`

**Contract** — a restrictor entity has spawned. Install its geometry under its own name,
replacing a placeholder if one was already handed out, and if its type says so, add it to
the level's default list of that sense. Notifies the subclass when a default list actually
changed. Called from the restrictor's spawn path.

```text
FUNCTION register_restrictor(restrictor, type)
  name = restrictor.name

  IF type names one of the default senses
    target = default_out or default_in per the type
    before = target
    target = normalize(target joined with name by a comma)
    IF target != before  on_default_restrictions_changed()

  shape = new shape wrapping restrictor       # builds its border immediately
  IF name IS NOT IN restrictions
    restrictions[name] = new bridge wrapping shape
  ELSE
    restrictions[name].change_implementation(shape)
```

**Invariants** — the order matters. The default list is updated *before* the geometry is
installed, and the notification it triggers re-derives every entity's restriction — which
resolves this name, which must therefore already... it does not: the resolution produces a
placeholder that goes inert, and the geometry installed a moment later swaps underneath it.
That is exactly the case the bridge exists for, and it is the reason the order is allowed
to be wrong.

The two default senses are mutually exclusive per restrictor: a restrictor is a default
permitted region or a default forbidden region or neither, never both. Anything else is a
hard failure.

**Notes** — the types distinguish *default* restrictors from ordinary ones, and within each
the permitted sense from the forbidden. An ordinary restrictor is named explicitly by the
entities it restricts; a default one restricts everybody on the level automatically. That
is how a level is fenced off at its edges without every spawn record naming the fence.

## `unregister_restrictor`

**Contract** — a restrictor entity is leaving. Remove its geometry, remove it from whichever
default list holds it (notifying on a change), and leave a placeholder behind under the same
bridge so that handles already given out stay valid and go inert. Hard-fails if the name
was never registered.

```text
FUNCTION unregister_restrictor(restrictor)
  name   = restrictor.name
  bridge = restrictions[name]                 # must exist
  remove name from restrictions

  IF name was removed from default_out
    on_default_restrictions_changed()
  ELSE IF name was removed from default_in
    on_default_restrictions_changed()

  bridge.change_implementation(new composition(self, name))
  restrictions[name] = bridge                 # same cell, empty geometry
  collect_garbage()
```

**Invariants** — the bridge is taken out and put back rather than left in place, because the
replacement must be built and installed atomically with respect to the registry — the map
entry and the cell's contents are two halves of one fact. The single-name composition it is
given never initializes, so every restriction naming this restrictor becomes inert. A
despawning restrictor therefore stops restricting rather than leaving a wall behind.

Removing a name from the default list searches both lists in order and stops at the first
hit, which is correct only because the senses are mutually exclusive.

## `collect_garbage`

**Contract** — destroy every registered bridge that holds a composition, has no handles
left, and has had none for five minutes of world clock. Shapes are exempt. Called on
registry misses and on unregistration.

```text
FUNCTION collect_garbage()
  FOR EACH (key, bridge) IN restrictions
    IF bridge.shape                                  CONTINUE   # owned by a live entity
    IF bridge still has handles                      CONTINUE
    IF now < bridge.last_release_at + 300000         CONTINUE
    destroy bridge ; remove key
```

**Invariants** — the delay is what makes this worth doing at all. Restriction combinations
churn: a creature's forbidden list changes as it takes and finishes jobs, and the same
combination comes back minutes later. Reclaiming on the last release would rebuild a border
— a scan over a region of the navigation mesh — every time. Five minutes is long enough to
outlive that churn and short enough that a level's worth of dead combinations does not
accumulate. The number is not derived from anything; it is a chosen hysteresis.

The release timestamp lives on the reference-counted base rather than in the registry, so
that both this collector and the one in
[`space_restriction_manager.cpp`](space_restriction_manager.cpp.md) read the same clock
from the same place.

## `clear`

**Contract** — destroy every bridge and empty both default lists. Called when the level
unloads.
