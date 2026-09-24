# src/xrAICore/Components/condition_state_inline.h

> The world state — a partial description of the world as an ordered set of property/value pairs, with an order-independent hash that makes state identity cheap enough to use as a planner vertex key.

**Needs** — [`condition_state.h`](condition_state.h.md) · [`operator_condition.h`](operator_condition.h.md)
**Used by** — [`condition_state.h`](condition_state.h.md) · [`operator_abstract_inline.h`](operator_abstract_inline.h.md) · [`problem_solver_inline.h`](problem_solver_inline.h.md)
**Tier floor** — T2: ordered-merge set algebra plus an incremental hash; no layout or latency constraint.

## Purpose

This is the planner's notion of "a world". Every operation in it exists to make one thing
fast: deciding whether two partial world descriptions are the same, because the planner's
search visits states and must recognise a state it has already expanded. Two decisions do
that work — the pairs are kept in ascending property order, so every comparison is a single
merge walk rather than a lookup per property; and the state carries a running hash that is
independent of insertion order, so unequal states are usually rejected without walking at all.

## State

```text
RECORD WorldState
  conditions : list<Property>   # ascending by property id, at most one entry per id
  hash       : int (32-bit)     # XOR of every entry's own hash; 0 when empty
```

where `Property` is a (property id, value) pair carrying its own hash — see
[`operator_condition_inline.h`](operator_condition_inline.h.md).

**Invariants**

- `conditions` is strictly ascending by property id. Every operation either preserves this
  or is only legal when the caller preserves it.
- `hash` equals the XOR-fold of the entries' hashes at all times. It is maintained
  incrementally: each insertion and each removal XORs the entry's hash in, and because XOR is
  its own inverse and is commutative, removal needs no recomputation and the result does not
  depend on the order entries were added.
- **A state is partial.** A property that does not appear is not "false"; it is unspecified.
  This is the single most consequential fact about the type: the planner's goal states,
  operator preconditions and operator effects are all partial, and every operation below is
  written around "absent means don't care".

## `add_condition` / `add_condition_back`

**Contract** — insert one property/value pair. The general form finds the insertion point by
binary search over the ordered list and requires that the property is not already present.
The `back` form appends and requires that the appended property is strictly greater than the
last — it is the fast path used by the operator-application routines, which build their
results by merging two already-ordered lists and therefore always produce ascending output.
A third form takes an insertion position the caller already computed, used when a caller is
mid-walk and would otherwise search twice.

**Invariants** — after either, the order holds and the hash has the new entry XORed in.

**Notes** — the duplicate check is an assertion, not a branch: inserting a property twice is a
programming error (the planner's world model has one value per property), not a condition to
recover from. A rebuild should keep it an error.

## `remove_condition`

**Contract** — drop the pair for a property, by binary search. Asserts the property is present.
XORs the removed entry's hash out.

**Notes** — the search key is built by pairing the property with a zero value. That works only
because the ordering compares property first and value second, so any value locates the same
position. A rebuild ordering purely on property id does not need the dummy value at all.

## `includes`

**Contract** — is every pair of the argument present in this state with the same value? One
merge walk; a property the argument names that this state does not have, or has with a
different value, answers no. Properties this state has that the argument does not name are
ignored.

```text
FUNCTION includes(other) -> bool
  walk this.conditions and other.conditions together in ascending property order
  WHEN this has a property other does not          -> skip it        # extra detail is fine
  WHEN other has a property this does not          -> RETURN false   # unmet requirement
  WHEN both have it with different values          -> RETURN false
  WHEN both have it with the same value            -> advance both
  RETURN true WHEN other is exhausted              # this may still have leftovers
```

**Notes** — this is the "does the current world satisfy this requirement" test, and its
asymmetry is the whole point: a state satisfies a partial requirement by matching it where it
speaks and being free everywhere else.

## `weight`

**Contract** — counts the properties the two states both mention and *disagree* on. Properties
only one of them mentions contribute nothing.

**Notes** — this is a distance between partial states, and it is the shape the planner's
heuristic wants. It is not used through this entry point: the declared parameter type says a
single property where the body requires a whole state, which compiles only because the routine
is never instantiated. The planner computes the same quantity itself. A rebuild should either
delete this or fix the parameter to be a state — it is dead as written.

## `operator-=` (difference)

**Contract** — reduces this state to just the pairs that the argument *contradicts*: properties
present in both where the values differ. Everything else — properties only one side mentions,
and properties both agree on — is dropped. The hash is rebuilt from the surviving entries.

**Notes** — the name reads like set subtraction and the behaviour is not that. It answers "in
what respects does the argument disagree with me", which is what a caller wanting to know what
still has to change asks for. Worth renaming in a rebuild.

## `operator<`

**Contract** — a total order over states: lexicographic over the ascending pair sequences,
comparing each pair by property first and value second; if one sequence is a prefix of the
other, the shorter is smaller.

**Notes** — exists so states can be sorted-container keys. It has no planning meaning and must
not be read as "less satisfied".

## `operator==`

**Contract** — equality. Compares hashes first and answers no immediately when they differ;
otherwise compares pair by pair, including the lengths. The hash is a filter, never the answer:
distinct states can share a hash, so the elementwise comparison is not optional.

## `hash_value`

**Contract** — the running hash. Constant time.

**Notes** — this is what the planner's visited-state table buckets on. The order-independence
is what makes it usable: the same set of pairs built by different operator sequences must land
in the same bucket or the search would re-expand states it had already seen.

## `property`

**Contract** — looks up one property by id and yields the pair, or nothing when the state is
empty of anything at or past that id.

**Notes** — this is a trap as written: the binary search returns the first entry whose property
is *at least* the requested one, and the routine only rejects the past-the-end case. When the
requested property is absent but a larger one exists, the caller receives that larger pair.
Every caller must compare the returned pair's property with the one it asked for. This surface
is exported to scripts, so the trap is reachable from game data; a rebuild should return
nothing on a mismatch instead.
