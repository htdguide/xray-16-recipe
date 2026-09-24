# src/xrGame/object_manager_inline.h

> The candidate-and-winner pattern every per-creature manager is built on: admit, score every frame, keep the minimum.

**Needs** — [`object_manager.h`](object_manager.h.md)
**Used by** — [`object_manager.h`](object_manager.h.md)
**Tier floor** — T3: a scored selection over a small list

## Purpose

This is the substance of the object manager, in a header because the type is generic over
what it manages. It is the shared skeleton of several of a creature's memory managers: each
holds a set of candidates of one kind — enemies, items, corpses, sound sources — and the
question each one answers is always "which of these is the one I should be acting on".

Its value to a rebuild is that it fixes *when* the choice is made and by what rule, so the
derived managers need only supply the score.

## State

```text
RECORD ObjectManager<T>
  objects  : list<T>          # the candidates
  selected : optional<T>      # this update's winner
```

**Invariants** — the winner is recomputed from scratch on every update and is valid only
between one update and the next. It is never incrementally maintained, so a candidate whose
score changed since the last update cannot make the selection stale.

The candidate list is *not* cleared by the update. It is cleared by reinitialization or by an
explicit reset, and grown by admission — so the owner is responsible for emptying it each
cycle if the candidate set is meant to be per-cycle. Most derived managers do exactly that.

## `add`

**Contract** — offers a candidate. Rejects it outright if it fails the usefulness test.
Otherwise adds it if it is not already present. Returns whether the candidate is now in the
set — so a duplicate reports success, and only a rejection reports failure.

```text
FUNCTION add(candidate) -> bool
  IF NOT is_useful(candidate) THEN RETURN false
  IF candidate is already present THEN RETURN true
  append candidate
  RETURN true
```

**Notes** — the presence test is a linear scan. The list is expected to hold a handful of
entries — the things one creature can currently perceive — so a scan is cheaper than any
index. A rebuild should keep it linear unless the candidate sets grow.

## `update`

**Contract** — rescores every candidate and selects the one with the **lowest** score.
Selects nothing when the list is empty. Does not remove anything.

```text
FUNCTION update()
  best := +infinity
  selected := none
  FOR EACH candidate IN objects
    score := do_evaluate(candidate)
    IF score < best
      best := score
      selected := candidate
```

**Invariants** — lower is better, and ties go to the **earliest** candidate in the list, since
the comparison is strict. Insertion order is therefore a tie-break, which makes the choice
stable across updates as long as the list order is stable — and the list order is insertion
order, which is perception order. A rebuild that reorders the list, or that uses a
non-strict comparison, will make a creature oscillate between two equally-good enemies.

**Notes** — every candidate is scored every update, with no caching. That is the deliberate
simplification: scores depend on distance, visibility and threat, all of which change
continuously, so a cache would be invalidated every frame anyway.

## `is_useful`

**Contract** — the default admission test: a candidate is admitted only if it carries the
spatial-registry flag marking it visible to AI. Everything else is rejected.

**Invariants** — this is where "the AI does not consider things it cannot in principle
perceive" is enforced once for every manager. A rebuild's equivalent flag must be set on the
same set of objects, or creatures will either ignore or fixate on things the original would
not.

**Notes** — the default begins with a cast of the candidate to the spatial interface and a
null check on the result, which cannot fail as written — the conversion is a reinterpretation,
not a checked one. It is dead defensive code from an earlier ownership model.

## `do_evaluate`

**Contract** — the scoring hook. The default returns zero for everything, which makes the
selection "the first candidate offered". Derived managers override it.

**Notes** — a default of zero rather than infinity means an unoverridden manager still selects
something rather than nothing. That is the friendlier failure, and it is why the base is
usable on its own for a "just give me any one of these" set.

## `reinit` / `reset`

**Contract** — `reinit` empties the candidate list and clears the winner. `reset` empties the
list but **leaves the winner set**.

**Invariants** — the asymmetry is real and is the reason both exist. A reset means "start
collecting candidates again this cycle" and the previous winner must remain readable while
the new set fills, so that a creature does not lose its target for the part of a frame
between reset and the first admission. A reinitialization means the creature is being rebuilt
and nothing should survive.

## `selected` / `objects` / `Load` / `reload`

**Contract** — `selected` returns the current winner or nothing; `objects` the whole candidate
list. The two configuration hooks are empty at this level and exist for derived managers to
fill.
