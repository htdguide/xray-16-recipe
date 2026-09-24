# src/xrGame/alife_spawn_registry_spawn.cpp

> The spawn walk: descending the authored spawn graph, rolling its edge weights, and deciding which records become entities.

**Needs** — [`alife_spawn_registry.h`](alife_spawn_registry.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a weighted recursive descent

## Purpose

The spawn graph says *what may exist*; this file turns it into *what does exist*. It is
called at the start of a game and again whenever the simulation decides to repopulate, and
it answers with a list of spawn-record identifiers to instantiate.

The shape the graph encodes:

- a **leaf** record is a concrete entity: spawn it;
- a record with edges is a **container or a chooser**. Each edge carries a weight, and the
  record's flags say whether the weights mean "roll each independently" (a container whose
  every item has its own chance) or "pick exactly one" (a chooser among alternatives).

Everything else here is the set of conditions under which a record is skipped.

## `fill_new_spawns(out spawns, game_time, existing)`

**Contract** — the entry point. Fills a list of spawn-record identifiers to instantiate,
given the current game time and the identifiers of records that have *already* produced a
live entity. Consumes randomness from the registry's stream. Output is sorted and
deduplicated.

```text
FUNCTION fill_new_spawns(OUT spawns, game_time, existing)
  sort and deduplicate existing      # the membership tests below need it sorted
  FOR EACH root IN spawn_roots
    descend(root, spawns, game_time, existing)
  sort and deduplicate spawns
```

**Invariants** — the walk starts only at roots (see
[`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md)), so a container's contents are
reached only through the container and with the container's odds applied.

The `existing` list is sorted up front because the "has this record already spawned"
test is a binary search over it, called once per visited record.

## `descend` — one record

**Contract** — decides whether a record spawns and, if it has children, how to treat them.

```text
FUNCTION descend(vertex, spawns, game_time, existing)
  IF NOT can_spawn(vertex.record, game_time, existing) -> RETURN

  IF vertex has no edges
    spawns.append(vertex.record.spawn_id)      # a leaf: this is a real entity
    RETURN

  IF vertex.record has the SINGLE_ITEM_ONLY flag
    descend_single(vertex, spawns, game_time, existing)   # choose exactly one child
    RETURN

  # Otherwise: roll each child independently against its own weight.
  FOR EACH edge IN vertex.edges
    IF random_unit_interval() < edge.weight
      descend(edge.target, spawns, game_time, existing)
```

**Invariants** —

- An interior record never spawns itself; only leaves become entities. A crate in the
  shipped data is therefore a leaf record *and* separately a chooser record if it has
  contents — the data authors model the crate and its contents as different vertices.
- In the independent mode an edge weight is a **probability in the unit interval**,
  compared directly against a uniform draw. In the choose-one mode (below) the same field
  is a **relative weight** summed across siblings. The same number means two different
  things depending on a flag on the parent, and nothing in the data marks which; a
  rebuild must read the flag before interpreting the weight.
- Zero children produced is a legitimate outcome of the independent mode: an empty crate.

## `descend_single` — choose exactly one child

**Contract** — picks one outgoing edge by weight and descends only into it.

```text
FUNCTION descend_single(vertex, spawns, game_time, existing)
  IF vertex.record has the IF_DESTROYED_ONLY flag AND subtree_already_spawned(vertex, existing)
    RETURN

  total = sum of edge weights
  draw  = random_real_below(total)
  IF draw >= total -> RETURN          # see Notes: currently unreachable

  running = 0
  FOR EACH edge IN vertex.edges
    running = running + edge.weight
    IF running > draw
      descend(edge.target, spawns, game_time, existing)
      RETURN
```

**Invariants** — the weights are relative, so authoring three alternatives at 1, 1 and 2
gives the last one half the outcomes. Weights need not sum to anything in particular.

**Notes** — the original multiplies the total and the running sum by a *group
probability*, a per-record chance that the chooser produces nothing at all. The field it
should read is commented out and replaced by the constant one, so the "produce nothing"
branch can never be taken and every chooser always produces exactly one child. A rebuild
implementing the field as intended changes shipped behaviour; implementing the constant
reproduces it.

## `can_spawn` — the four gates

**Contract** — a record may spawn when it is enabled, and is not stopped by the count
limit, the time limit or the already-exists limit.

```text
FUNCTION can_spawn(record, game_time, existing) -> bool
  RETURN enabled(record)
     AND NOT count_limited(record)
     AND NOT time_limited(record, game_time)
     AND NOT existence_limited(record, existing)
```

Each gate:

- **enabled** — the record's own flag. This is the designer's switch and the only gate
  that behaves as its name suggests.
- **count limited** — a record without the *infinite count* flag is limited. The
  comparison against a per-record maximum and a running count is present in the source but
  disabled, so in practice a record is blocked unless it is flagged infinite.
- **time limited** — a record without the *surge only* flag is limited. The comparison
  against a per-record next-spawn time is likewise disabled, so a record is blocked unless
  it is flagged surge-only.
- **existence limited** — a record flagged *if destroyed only* is blocked while an entity
  from it still exists.

**Invariants and Notes** — taken together, the two disabled comparisons mean the
**respawn machinery is effectively off**: on the second and later calls, a record passes
only if it is flagged both infinite-count and surge-only. That is not an accident of
reading — the per-record count and next-spawn-time fields exist in the record, are
serialized, and are never compared. The shipped games repopulate the world through
script-driven respawners rather than through this mechanism, which is presumably why it
was disabled.

A rebuild has a choice, and should make it deliberately: reproduce the disabled behaviour
(and then the count and time fields are dead weight in the format), or implement the
comparisons as the source clearly intended and accept that the shipped data's spawn
records will begin respawning in ways the original never did. The recipe cannot recover
which was meant; the commented-out lines say what the code *would* do, not what the game
*should* do.

## `subtree_already_spawned`

**Contract** — has this record, or any record reachable from it, already produced a live
entity?

```text
FUNCTION subtree_already_spawned(vertex, existing) -> bool
  IF vertex has no edges
    RETURN existing contains vertex.record.spawn_id
  FOR EACH edge IN vertex.edges
    IF subtree_already_spawned(edge.target, existing)
      RETURN true
  RETURN false
```

**Invariants** — an interior record is considered "already spawned" if **any** leaf under
it is live. That is what makes *if destroyed only* work for a container: the crate is not
re-offered while any of its possible contents still exists in the world, regardless of
which alternative was chosen last time.

The leaf test is a binary search, which is why the caller sorts the existing list first.
