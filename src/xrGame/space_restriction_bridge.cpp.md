# src/xrGame/space_restriction_bridge.cpp

> The indirection cell every restriction is held through, so that a named restrictor can be swapped from a not-yet-spawned placeholder to real geometry without invalidating anybody's handle — plus the two boundary tests that need the border's spatial sort order.

**Needs** — [`space_restriction_bridge.h`](space_restriction_bridge.h.md) · [`space_restriction_bridge_inline.h`](space_restriction_bridge_inline.h.md) · [`space_restriction_base.h`](space_restriction_base.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a binary search over a sorted vertex list plus delegation

## Purpose

Restrictor entities spawn and despawn while restrictions that name them are already in
flight. If a restriction held the geometry directly, every such transition would have to
find and patch every holder. Instead everything holds a *bridge*: a reference-counted cell
with one replaceable implementation inside it. Registering a restrictor swaps the cell's
contents from a placeholder to real geometry; unregistering swaps it back. Holders never
notice.

The cell also owns its implementation's lifetime, and it carries the release timestamp the
garbage collectors use, so the two ideas — stable identity and deferred reclamation — sit
in the same object.

## State

```text
RECORD RestrictionBridge
  implementation  : restriction        # a shape or a composition; never absent
  last_release_at : int                # world clock at the moment the last handle let go
```

**Invariants** — the implementation is never absent: a bridge is created with one and
replaced with another, never emptied. Replacing destroys the old one, so no handle may be
held to the implementation itself — only to the bridge. That is the rule the whole
subsystem rests on, and it is why nothing outside this file names a shape or a composition
directly.

## `change_implementation`

**Contract** — destroy the current implementation and install a new one. Called exactly
twice per restrictor entity lifetime: once when it spawns (placeholder composition to real
shape) and once when it despawns (shape back to placeholder). Not reentrant; no handle to
the old implementation may outlive the call.

**Notes** — the despawn direction is the interesting one. The replacement placeholder names
a single restrictor that no longer exists, and such a composition never reaches the
initialized state — so every restriction built on it stays inert and answers *accessible*
to everything. A despawning restrictor therefore stops restricting, rather than leaving a
dangling barrier or crashing a search in flight.

## `on_border`

**Contract** — is this world position's own navigation vertex a border vertex? Hard-fails
if the position is not over the navigation mesh. Reads the level graph; allocates nothing.

```text
FUNCTION on_border(position) -> bool
  key = packed horizontal grid coordinate of position
  i = lower_bound(border, key)            # border is sorted by exactly this key
  IF i is past the end OR key_of(border[i]) != key  RETURN false

  v = vertex_at(position)
  IF v is not a valid vertex  RETURN false

  # Several border vertices can share one horizontal cell — stacked floors,
  # a walkway over a road — so scan the whole run of equal keys.
  WHILE i is in range AND key_of(border[i]) == key
    IF border[i] == v  RETURN true
    i = i + 1
  RETURN false
```

**Invariants** — this is the consumer that makes the border's horizontal-position sort a
contract rather than a tidiness measure. A border left in identifier order makes the binary
search return nonsense, and the failure is silent: positions stop being recognized as
on-border and bodies are allowed to stand inside walls.

**Notes** — the two-stage lookup — narrow by horizontal cell, then match the exact vertex —
is what lets one sorted list answer a 3-D question. The alternative, a set keyed by vertex
identifier, would cost a second structure per restriction and buy nothing, because the
other consumer of the border wants it in this order anyway.

## `out_of_border`

**Contract** — is this position outside the volume, judged at the height of its own
navigation cell rather than at the height it was given? Hard-fails if the position is not
over the navigation mesh; answers *true* for a position over no valid vertex.

```text
FUNCTION out_of_border(position) -> bool
  v = vertex_at(position)
  IF v is not a valid vertex  RETURN true        # off the mesh counts as outside

  probe = sphere at (position.x, plane_height(v, position.x, position.z), position.z)
          with a near-zero radius
  RETURN NOT inside(probe)
```

**Notes** — snapping the probe to the cell's own sloped plane is the whole point. A
position handed in from a path or a script carries a height that may float above or sink
below the walkable surface; testing it as given would call a point inside a volume that a
body standing there would not be in, or the reverse on a slope. This is the same snapping
the per-vertex containment test uses, applied to one arbitrary point.

The near-zero radius rather than zero is the same concession as everywhere else in the
family: a degenerate sphere breaks the volume tests.

## Delegations

**Contract** — `border`, `initialized`, `initialize`, `name`, `shape`, `default_restrictor`,
`sphere`, and the three `inside` forms each forward to the current implementation and
nothing more. `accessible_nearest` forwards to the generic search with the implementation
as its own subject.

**Notes** — each `inside` form is wrapped in a named profiler scope, which is the only
reason they are written out rather than generated. The profiling is incidental; the
forwarding is not, because it is the indirection that makes the swap invisible.
