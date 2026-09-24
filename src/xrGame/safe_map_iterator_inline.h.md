# src/xrGame/safe_map_iterator_inline.h

> The round-robin walk: advance the cursor past an entry *before* updating it, so that an entry which removes itself mid-update cannot invalidate the walk.

**Needs** — [`safe_map_iterator.h`](safe_map_iterator.h.md)
**Used by** — [`safe_map_iterator.h`](safe_map_iterator.h.md)
**Tier floor** — T2: a stable cursor into an ordered map, a wall clock, and a cycle counter

## Purpose

Holds the two ideas the type exists for: how a walk survives its own members mutating the
collection, and how a walk that cannot finish in one frame resumes fairly.

## State

```text
RECORD SafeMapIterator
  objects          : map<key, reference to value>   # ordered; iteration order is key order
  cursor           : position in objects            # the NEXT entry to update; wraps
  cycle_count      : int (64-bit)                   # passes started, handed to each update
  timer            : wall clock, restarted per pass
  max_process_time : real (seconds)                 # the per-pass budget
  first_update     : bool                           # this pass is exempt from the budget
```

**Invariants**

- The cursor points at the **next** entry, never at the current one. Everything else follows
  from that.
- The cursor is always valid: empty means it equals the end, non-empty means it addresses an
  entry. `add` and `remove` each restore this, which is why they cannot be plain container
  operations.
- Iteration order is the key order, and it is stable. A newly added entry lands wherever its
  key sorts, which may be before or after the cursor — so a member added mid-pass may be
  updated in that same pass or in the next one, and neither is a bug. A consumer that needs
  "not until next pass" must check the cycle counter itself.
- The cycle counter is 64 bits wide *because* the update predicate keys per-member scheduling
  off it. Wrapping it would make a member believe a pass it already handled is new.

## `update`

**Contract** — runs one pass. Calls the caller's predicate for each entry in cursor order,
stopping when the budget is exhausted, when the predicate declines, or when the collection
runs out. Answers how many entries it updated. Does nothing and answers zero on an empty
registry. Blocks for up to the budget, plus however long the last entry's update takes —
**the budget is checked between entries, never inside one**, so one slow member overruns it.

```text
FUNCTION update(predicate, exempt_next_pass_from_budget) -> int
  IF objects is empty THEN RETURN 0

  timer.restart()
  cycle_count = cycle_count + 1
  count = 0

  entry = cursor
  WHILE entry is valid
        AND NOT budget_exhausted()
        AND predicate(entry, cycle_count, ASKING = true)    # may this entry be updated now?
    advance_cursor()          # BEFORE the update: the entry may remove itself, and the
                              #   cursor must already be somewhere else when it does
    predicate(entry, cycle_count)                            # the real update
    count = count + 1
    entry = cursor

  first_update = exempt_next_pass_from_budget
  RETURN count
```

**Invariants**

- **The cursor advances before the update runs.** This is the whole point of the type. An
  entry's update is allowed to remove that entry — a creature dying during its own update is
  the common case — and if the cursor still pointed at it, the walk would resume from a
  removed position. Advancing first means the removal path finds the cursor elsewhere and
  leaves it alone.
- The predicate is called **twice per entry**: once as a question ("should this one be
  updated this pass?") and once as the update. The question form is how consumers implement
  per-member rate limiting against the cycle counter without the iterator knowing anything
  about rates. A *false* answer stops the pass entirely rather than skipping that entry —
  so a consumer using it as a filter would starve everything after the first decline. It is
  a stop condition, not a filter, and reading it as a filter is the easiest mistake here.
- The budget exemption is consumed at the *end* of the pass and set from the caller's
  argument, not cleared. So the caller decides, per pass, whether the *next* pass runs
  unbudgeted. The exemption exists for load time, when the whole registry must be brought
  up to date before the first frame is presented and a frame budget is meaningless.

## `advance_cursor`

**Contract** — moves the cursor to the next entry, wrapping to the first at the end and
resetting to the end marker when the registry is empty. The wrap is what makes the walk
round-robin rather than a single pass.

## `add`

**Contract** — registers a value under a key. Refuses a duplicate key — loudly by default,
silently in the relaxed mode — and leaves the registry untouched when it does. When the
registry was empty, the cursor is re-seated at the first entry, since a cursor sitting at the
end marker would otherwise never advance into the newly non-empty collection.

## `remove`

**Contract** — withdraws a key. Refuses an absent key, loudly or silently. **If the cursor
points at the entry being removed, the cursor is advanced first.** Removing the last entry
re-seats the cursor at the end marker. This is the other half of the mid-walk safety
property: removal from inside a walk is explicitly supported, and removal from outside a
walk goes through the same path.

## `clear`

**Contract** — empties the registry by repeatedly removing the first entry, rather than by
dropping the container. Going through `remove` keeps the cursor maintenance in one place; the
cost is quadratic-ish behaviour on a large registry, which nothing in the engine hits because
clearing happens at level teardown.

## `begin`

**Contract** — seats the cursor at the first entry and exempts the next pass from the budget.
The "start over, completely, now" operation, used when the registry's contents have been
wholesale replaced — a level load or a save restore — and a partial pass would leave half the
members holding stale state.

## `set_process_time` · `objects` · `empty`

**Contract** — set the per-pass budget in seconds; read the registry; ask whether it is
empty.

**Notes** — the budget is a duration, not a count. Two consequences a rebuild should expect:
the number of entries updated per frame varies with machine speed, so behaviour is not
reproducible across machines from this path; and a registry whose members are individually
slower than the budget degenerates to one member per pass without any warning. Both are
accepted in the original. A rebuild wanting determinism — which the conformance criteria ask
for in physics but not here — must budget by count instead and accept the frame-time
variance.
