# src/xrGame/obstacles_query_inline.h

> The obstacle set's lazy-recompute protocol: what invalidates the cached area, and what forces it back.

**Needs** — [`obstacles_query.h`](obstacles_query.h.md)
**Used by** — [`obstacles_query.cpp`](obstacles_query.cpp.md) · [`obstacles_query.h`](obstacles_query.h.md)
**Tier floor** — T3: cache invalidation over a set

## Purpose

Split from the class body by the original language's rules, but not incidental: this is where
the query's freshness discipline is defined, and that discipline is the only thing standing
between the pathfinder and a per-frame recomputation of every creature's blocked-vertex set.

## State

Operates on the record declared in [`obstacles_query.h`](obstacles_query.h.md): the object set,
the cached vertex area, the checksum, and the freshness flag.

## The freshness protocol

**Contract** — three rules, and they are the page:

1. **Adding an object clears the freshness flag.** Nothing is recomputed at that moment.
2. **Reading the area recomputes it if the flag is clear**, and sets the flag.
3. **Everything else** — the checksum, the object set, the flag itself — reads the cached
   value without recomputing.

```text
FUNCTION add(object)
  IF object is already present THEN RETURN      # no invalidation for a duplicate
  actual := false
  insert (object, sentinel checksum)

FUNCTION area() -> list<vertex>
  IF NOT actual THEN compute_area()
  RETURN cached area
```

**Invariants** — the duplicate check must precede the invalidation. Re-adding an object a
creature already knows about is the common case — the avoidance system re-offers the same
obstacles every frame — and invalidating on it would defeat the whole cache.

Each object is stored with a per-object checksum, initialized to the all-ones sentinel so that
a freshly added object never matches whatever its real checksum turns out to be, and is
therefore always seen as changed on the first comparison.

The area accessor exists in a reading form that forwards to the mutating one, because reading
the area may legitimately mutate the cache. A rebuild in a language that distinguishes these
must either make the cache mutable or recompute eagerly.

## `refresh_objects` / `update_objects`

**Contract** — `refresh_objects` forces a recomputation and answers whether the checksum
changed as a result. `update_objects` does that only if a proximity test says something
relevant moved, and otherwise answers false without recomputing.

```text
FUNCTION refresh_objects() -> bool           # "did anything actually change"
  actual := false
  before := checksum
  compute_area()
  RETURN checksum != before

FUNCTION update_objects(position, radius) -> bool
  IF objects_changed(position, radius) THEN RETURN refresh_objects()
  RETURN false
```

**Invariants** — the return value is "the *result* changed", not "a recomputation happened".
Callers use it to decide whether to throw away a computed path, and a recomputation that lands
on the same answer must not cost them a path. That is why the checksum is compared rather
than the recomputation being reported.

## `clear` / `swap` / `copy`

**Contract** — `clear` empties both the object set and the area and resets the checksum to
zero and the flag to fresh. `swap` exchanges all four fields with another query. `copy`
assigns all four.

**Invariants** — an empty query is **fresh**, not stale: its area is correctly empty and
recomputing would be wasted work. A rebuild initializing the flag to stale makes every
creature that has no obstacles recompute an empty set every frame.

`swap` and `copy` carry the flag and the checksum along with the data, so a query's freshness
travels with it. Copying a stale query and reading its area recomputes in the copy and leaves
the original stale, which is correct but means the work is done twice; callers are expected to
refresh before copying.

## Equality

**Contract** — two queries are equal when their checksums match **and** their vertex areas
match. The object sets are not compared.

**Invariants** — comparing areas rather than objects is the point: two different sets of
objects that block the same vertices are equivalent to the pathfinder, and treating them as
equal avoids a needless replan. The checksum is checked first as a cheap rejection.

**Notes** — the object-set comparison is present and commented out, which is what makes the
equality an *effect* comparison rather than a *cause* comparison. A rebuild should keep it
that way.
