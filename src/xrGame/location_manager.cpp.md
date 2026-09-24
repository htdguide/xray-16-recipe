# src/xrGame/location_manager.cpp

> Where a creature is willing to be: the terrain masks the alife simulation scores cross-level graph vertices against when moving it off-screen.

**Needs** — [`location_manager.h`](location_manager.h.md) · [`GameObject.h`](GameObject.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: reads configuration into a preference list

## Purpose

While a creature is offline, the alife simulation moves it around the game graph, and the
choice of where to send it is a preference over *terrain*: a bloodsucker belongs in
basements and swamps, a stalker on roads and in camps. Each game graph vertex carries a
terrain descriptor; this file holds the creature's side of the match — a list of masks, each
saying "terrain like this, weighted like this".

It is a separate, tiny file because the preference is owned per creature but its parsing —
the mask syntax and its resolution against named terrain sections — belongs to the
navigation layer, and this file is only the binding between them.

## State

```text
RECORD LocationManager
  owner         : GameObject
  terrain_masks : list<TerrainMask>   # empty means: no preference, all terrain equal
```

**Invariant** — an empty list is a valid, meaningful state and not an uninitialized one. The
alife simulation treats a creature with no masks as indifferent to terrain rather than
refusing every vertex. This is what lets non-creature entities carry the manager harmlessly.

## `Load`

**Contract** — reads the preference from a configuration section, once, at spawn. Looks for a
`terrain` key in the section: if present, its value names where the masks live; if absent,
**the section itself** is used as that name. Blocks; allocates.

The fallback is the load-bearing part. It means a creature's own section can either point at
a shared terrain profile — many creatures of a family sharing one — or *be* the profile,
with the masks written inline. Both forms appear in the shipped data and a rebuild must
support both.

## `reload`

**Contract** — the per-*instance* override, applied after `Load`. An individual spawned
entity may carry its own configuration block, and if that block declares an `alife` section
with a `terrain` key, that terrain profile **replaces** the class-level one. Does nothing —
leaving the loaded preference intact — when the entity carries no such block. This is how a
level designer pins one particular mutant to one particular kind of ground without creating
a new class.

**Invariant** — the override replaces rather than merges. A partially-specified instance
override would otherwise inherit weights from the class and produce a blend no author asked
for.

## `clear_location_types` · `add_location_type`

**Contract** — build the list imperatively instead of from a section: empty it, then append
one mask at a time from a mask string. These exist for script and for smart terrains, which
compute a creature's acceptable ground from the job they are assigning rather than from the
creature's own data. Appending does not deduplicate; a repeated mask is counted twice in the
alife simulation's weighting, which is a usable way to emphasize a terrain and is also an
easy accident.
