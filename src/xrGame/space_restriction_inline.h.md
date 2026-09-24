# src/xrGame/space_restriction_inline.h

> Stamps and clears the merged border on the level graph, and answers containment in the permitted space with the strictness flag inverted between the two senses.

**Needs** — [`space_restriction.h`](space_restriction.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`space_restriction.cpp`](space_restriction.cpp.md) · [`space_restriction.h`](space_restriction.h.md)
**Tier floor** — T2: two mask operations on the navigation mesh and one composed predicate

## Purpose

Three of the restriction's operations are here rather than in the implementation file
because one of them is generic over the argument pair that describes an entity's intended
movement. The split is an artifact; a rebuild should fold all of this into the type. What
is not an artifact is the pairing: stamping and clearing the border are exact inverses and
must be read together.

## `add_border`

**Contract** — stamp this restriction's border onto the level graph as a search barrier, so
that a path search run afterwards cannot cross it. Takes two arguments describing the
movement about to be planned — a start and a destination, as positions or as vertices —
which the shipped build ignores. No-op on an uninitialized restriction. Hard-fails if the
border is already stamped.

```text
FUNCTION add_border(start, destination)
  IF NOT initialized  RETURN              # inert restriction: nothing to stamp
  REQUIRE NOT applied
  applied = true

  IF out EXISTS
    level_graph.set_mask(border)          # the merged boundary of out MINUS in
    RETURN

  level_graph.set_mask(in.border)         # nothing to be inside of: the forbidden rim alone
```

**Invariants** — `applied` gates the pair. The border may be stamped once and must be
cleared before it is stamped again, and the manager refuses to change an entity's
restriction while it is stamped. The mask is global state on the level graph shared by
every search, so a leaked stamp is a permanent invisible wall.

**Notes** — the start and destination arguments are the interface to the lazily-enabled
policy described in [`space_restriction.cpp`](space_restriction.cpp.md): with that policy
on, each forbidden cluster is tested against the intended movement and only the relevant
ones are stamped. With it off — as shipped — the arguments are dead and a rebuild may drop
them, at the cost of not being able to restore the optimization without changing the
signature again.

## `remove_border`

**Contract** — the exact inverse: clear the same mask that `add_border` set. No-op on an
uninitialized restriction. Hard-fails if nothing is stamped.

**Invariants** — it must clear *the same list*, which is why the border is cached rather
than recomputed and why the restriction may not be swapped while stamped.

## `inside`

**Contract** — is a sphere, or a navigation vertex, within the permitted space? The sphere
form is exactly the accessibility test. The vertex form composes the two members with
opposite strictness.

```text
FUNCTION inside(vertex, partially) -> bool
  RETURN (out IS none OR out.inside(vertex, partially))
     AND (in  IS none OR NOT in.inside(vertex, NOT partially))
```

**Invariants** — the flag is **inverted** on the forbidden side. Asking "is this cell fully
within the permitted space" means asking the permitted volume for full containment *and*
the forbidden volume for not even partial containment. Asking for partial permitted
containment means asking the forbidden volume for not-full containment. A rebuild that
passes the same flag to both members produces a space that is subtly too large in one
direction and too small in the other, and the symptom is creatures clipping into zones
rather than a visible failure.

## Constructor and accessors

**Contract** — construction takes the owning manager and the two name lists verbatim and
records them; nothing is resolved and no border is built until the first query. The
accessors return the two name lists, the initialized flag and the applied flag.

**Notes** — deferring all work to the first query is what lets restrictions be constructed
during level load, before the restrictor entities they name have spawned.
