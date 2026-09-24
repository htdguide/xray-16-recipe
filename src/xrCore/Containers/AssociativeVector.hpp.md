# src/xrCore/Containers/AssociativeVector.hpp

> A sorted-array map: the lookup shape of a dictionary with the memory shape and iteration cost of a list. Used wherever a map is small, read far more than written, and walked in key order.

**Needs** — [`AssociativeVectorComparer.hpp`](AssociativeVectorComparer.hpp.md) · [`../xrCore.h`](../xrCore.h.md) · [`../../xrCommon/xr_vector.h`](../../xrCommon/xr_vector.h.md)
**Used by** — [`editor_environment_levels_manager.hpp`](../../editors/xrWeatherEngine/editor_environment_levels_manager.hpp.md) · [`pch.h`](../../utils/mp_balancer/pch.h.md) · [`statistics_collector.hpp`](../../utils/mp_balancer/statistics_collector.hpp.md) · [`pch.h`](../../utils/mp_configs_verifyer/pch.h.md) · [`problem_solver.h`](../../xrAICore/Components/problem_solver.h.md) · [`patrol_path_storage.h`](../../xrAICore/Navigation/PatrolPath/patrol_path_storage.h.md) · [`game_graph_space.h`](../../xrAICore/Navigation/game_graph_space.h.md) · [`AssociativeVectorComparer.hpp`](AssociativeVectorComparer.hpp.md) · [`Message_Filter.h`](../../xrGame/Message_Filter.h.md) · [`detail_path_manager.h`](../../xrGame/detail_path_manager.h.md) · [`file_transfer.h`](../../xrGame/file_transfer.h.md) · [`obstacles_query.h`](../../xrGame/obstacles_query.h.md) · [`profile_data_types.h`](../../xrGame/profile_data_types.h.md) · [`purchase_list.h`](../../xrGame/purchase_list.h.md) · _and 2 more_
**Tier floor** — T2: an ordered sequence and a binary search. Nothing here needs manual memory; the reason it exists at all is a cache-locality argument that survives into T2 and mostly evaporates at T3.

## Purpose

The engine uses this in roughly a hundred places where the obvious choice would be a node-based map: inventory slots, dialogue tables, per-team scoreboards, sound-player channels, spawn registries. The decision behind every one of them is the same and it is worth stating once: **these maps hold a handful to a few hundred entries, are built once and read constantly, and are frequently iterated in full.** A node-based map pays a pointer chase per element for both lookup and iteration and an allocation per insert; a sorted array pays a memmove per insert and nothing else.

The cost is inserts, which are linear in the size of the map. That is the trade, and it is why this type must not be substituted for a map that grows during play.

## State

```text
RECORD AssociativeVector<K, V>
  entries : list<pair<K, V>>       # invariant: strictly ascending by key,
                                   # no duplicate keys, at every observable moment
```

**Invariants** — the ordering invariant is the whole type. Every read operation is a binary search that assumes it; every write is responsible for restoring it before returning. There is no lazy-sort state despite appearances — see the notes.

## `find` / `lower_bound` / `upper_bound` / `equal_range` / `count`

**Contract** — binary search by key. `find` returns the entry or the end marker; `lower_bound` returns the first entry not ordered before the key; `upper_bound` the first ordered after it. `equal_range` returns a range of length zero or one, never more, because keys are unique. `count` is zero or one. None of these allocate; all are logarithmic in the entry count.

**Notes** — `find` is written as a lower-bound followed by one inequality test rather than as its own search. That is not an optimization, it is the standard way to get "found or not" out of a bound: the bound lands on the first key not less than the target, so the target is present exactly when that key is also not greater.

## `insert` / `emplace`

**Contract** — locate the insertion point by binary search; if a matching key is already present, **overwrite its value** and report "not inserted"; otherwise splice a new entry in at the bound and report "inserted". Returns the position either way. Invalidates every outstanding position into the container on a genuine insert.

**Invariants** — the overwrite-on-collision behaviour is the one place this type differs meaningfully from the map it substitutes for, whose insert leaves the existing value alone. Call sites that rely on the standard behaviour silently change meaning. A rebuild should pick one and name the operation for it — `put` for overwrite, `insert_if_absent` for the other.

## `insert(first, last)` — bulk

**Contract** — insert a whole range. Chooses between two strategies by size.

```text
FUNCTION insert_range(first, last)
  incoming := count(first, last)
  IF incoming < log2(size + incoming)
    insert each one individually        # each is a search plus a splice
  ELSE
    append them all, then sort the whole thing
```

**Notes** — the threshold is a genuine decision and the comparison is the right shape: inserting one element costs a search plus a linear splice, so *n* individual inserts cost about *n* times the size; appending and re-sorting costs about size times log(size). The crossover is where *n* is comparable to log(size), which is what the test says. The *implementation* of the test is sloppy — it compares a signed difference against a floating-point logarithm — but the decision it encodes is sound, and a rebuild should keep the decision and fix the arithmetic.

Re-sorting the whole container rather than merging the two sorted runs is the obvious missed optimization; a merge would be linear. Nothing in the source suggests it was considered.

## `insert(where, value)` — positional hint

**Contract** — nominally a hinted insert: if the hint is exactly right, splice there; otherwise fall through to the searching insert.

**Notes** — **the hint test in the source cannot be satisfied.** It requires simultaneously that the hint is not the end, that the hint precedes the value, that the hint's offset equals the container's size (which for a non-end position is impossible), and two further conditions on the element after it that contradict each other. Every call therefore takes the fall-through path. The behaviour is correct; the fast path is dead.

A rebuild should either write the hint check properly — hint is a valid position, the value belongs strictly between the element before it and the element at it — or drop the overload. Since nothing in the engine depends on the fast path (it has never executed), dropping it is safe.

## `erase`

**Contract** — three forms: erase at a position, erase a range, erase by key. The key form searches, removes if found, and reports how many entries went away (zero or one). All are linear in the elements after the removal point. Positions after the removal are invalidated.

## `operator[]`

**Contract** — find the key; if absent, insert it with a default-constructed value; return a reference to the value. Inserting invalidates outstanding positions, which makes the common `m[a] = m[b]` shape hazardous in a way the map it replaces shares.

## Whole-container operations

**Contract** — `clear`, `reserve`, `size`, `empty`, `max_size`, `swap` and the six relational comparisons all delegate to the underlying sequence with no added logic. The relational comparisons are lexicographic over the sorted entries, which — because the ordering invariant holds — makes them a well-defined order on maps rather than on insertion histories.

**Notes** — `reserve` is the operation that makes this type worth using: a map whose final size is known is built with one allocation and no reallocation, and the splice cost of building it in key order is then pure memmove. Call sites that know their size and do not reserve are giving up most of the benefit.

## `begin` / `end` / `rbegin` / `rend`

**Contract** — iteration in ascending key order, forward or reverse, over contiguous storage.

**Notes** — every one of these, and every search, first calls an internal *actualize* step. **That step is empty.** It is the vestige of a design where the container deferred sorting until the first read — build unsorted, sort once, then serve searches — which would have made bulk construction linear. The hook survives in the shape of the code; the deferral does not. A rebuild has a real choice here: implement the deferral the hook was left for (a dirty flag, sorted on first read), or delete the hook. What it must not do is keep the hook empty and assume a lazy-sort invariant that does not exist.

One genuine bug rides on the vestige: the reverse-end accessor is declared to return a forward position rather than a reverse one. It compiles because both are pointers into the same array and is wrong for any reverse traversal that consults it. Nothing in the engine does.

## Notes on the whole file

The container derives privately from both the sequence and the comparison object. Both are incidental: the first is composition written as inheritance to get the sequence's member types for free, and the second is a C++ idiom for storing a stateless comparison at zero cost. A rebuild composes a sequence and a comparison function and loses nothing.

The type deliberately exposes only part of a map's surface — there is no multi-key form, no node handle, no hint that works, and no bidirectional erase-by-value. That narrowness is a feature: everything it does expose is cheap on a sorted array, and the operations it omits are the ones that are not.
