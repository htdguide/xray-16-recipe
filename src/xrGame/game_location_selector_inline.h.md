# src/xrGame/game_location_selector_inline.h

> The random walk across the game graph: count the neighbours that match the creature's terrain preference, pick one uniformly, and never step straight back where you came from.

**Needs** — [`game_location_selector.h`](game_location_selector.h.md) · [`location_manager.h`](location_manager.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md)
**Used by** — [`game_location_selector.h`](game_location_selector.h.md)
**Tier floor** — T3: graph traversal and a uniform choice

## Purpose

Carries the behaviour of the game-graph location selector declared in
[`game_location_selector.h`](game_location_selector.h.md). The split is an artifact of the
original's template mechanics; a rebuild writes one unit.

## `select_location`

**Contract** — dispatches on the strategy.

```text
FUNCTION select_location(from) -> to
  IF strategy is mask THEN
    IF this selector is in use THEN run the inherited search from `from`
    ELSE clear the failure flag          # nothing to search for is not a failure
  ELSE IF strategy is random_branching THEN
    IF a graph is bound THEN to = select_random(from)
    failed = failed AND (to == from)     # only a choice that went nowhere is a failure
```

**Invariants** — under random branching, failure is redefined: an inherited failure that
nonetheless moved the creature is not a failure, because a random walk has no goal to miss.
Only standing still counts.

## `select_random_location`

**Contract** — chooses uniformly among the neighbours of the current vertex that pass three
filters and match at least one of the creature's preferred terrain masks. Counts first, then
picks, then walks again to the chosen one.

```text
FUNCTION select_random(from) -> to
  IF previous is invalid THEN previous = from   # first step: treat ourselves as the origin

  # Pass one: count the candidates.
  count = 0
  FOR EACH neighbour n OF from
    IF n == previous THEN CONTINUE              # do not double back
    IF n is not on the currently loaded level THEN CONTINUE
    IF NOT accessible(n) THEN CONTINUE
    FOR EACH preferred mask m OF the location manager
      IF n's terrain types match m THEN count = count + 1

  IF count == 0 THEN
    # Nowhere to go: go back, or stay.
    IF from != previous AND accessible(previous) THEN to = previous
    ELSE to = from
  ELSE
    choice = uniform integer in [0, count]
    # Pass two: walk the same order and take the choice-th match.
    FOR EACH neighbour n OF from, same three filters
      FOR EACH preferred mask m
        IF n matches m THEN
          IF the running index != choice THEN advance the index AND CONTINUE
          to = n; STOP both loops

  previous = from
```

**Invariants**

- **A vertex is counted once per matching mask, not once per vertex.** A neighbour matching
  three of the creature's preferences gets three chances to be drawn. Whether that is
  intended weighting or an oversight cannot be recovered from the source; the effect is that
  terrain a creature likes several ways attracts it more strongly, which is at least
  defensible. Reproduce it — the shipped terrain masks were tuned against this behaviour.
- **The "on the current level" filter means this only ever chooses within the loaded level.**
  The game graph spans levels, but a creature stepping across a level boundary is handled
  elsewhere; this selector deliberately keeps the wanderer where the player can meet it.
- The previous-vertex memory is updated to the *start* vertex at the end, unconditionally —
  including on the paths that found nothing. So a creature that had to turn back records the
  place it turned back from, and will not immediately turn back again.
- The two passes must apply identical filters in an identical order, or the index drawn in
  the first pass names a different vertex in the second. This is the classic reservoir-free
  "count then take the n-th" and a rebuild that collects candidates into a list in one pass
  is simpler and equivalent.

**Notes** — the random draw is over the closed range from zero to the count, so the highest
value is one past the last candidate and the second pass then finds nothing, leaving the
destination unchanged from whatever the caller passed in. That is an off-by-one: the range
should be half-open. Its effect is a small chance per decision of not moving, which is
invisible in play and is why it survived. A rebuild should use a half-open range.

## `actual`

**Contract** — under random branching, a choice stays actual until the path to it is
completed; there is nothing to re-evaluate, because there was no goal. Under the mask
strategy, defers to the inherited test, which re-checks whether the destination still scores
well from where the creature now stands.

## `accessible`

**Contract** — whether the creature's restrictors permit the level vertex underlying a game
graph vertex. With no restricted object, everything is accessible.

**Invariants** — the test descends from the coarse graph to the fine one: a game graph vertex
carries the level vertex it corresponds to, and restrictors are expressed over level
vertices. The two graphs are joined at exactly this point.
