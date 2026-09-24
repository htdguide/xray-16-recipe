# src/xrCore/Containers/AssociativeVectorComparer.hpp

> Lifts a key ordering to an ordering over key/value pairs, so one comparison serves both the sort and the searches.

**Needs** — [`AssociativeVector.hpp`](AssociativeVector.hpp.md)
**Used by** — [`AssociativeVector.hpp`](AssociativeVector.hpp.md)
**Tier floor** — T3: it is one comparison, applied to whichever of its arguments happens to be a pair.

## Purpose

The sorted-array map in [`AssociativeVector.hpp`](AssociativeVector.hpp.md) sorts *pairs* but searches by *key*. A binary search over pairs given a bare key therefore needs a comparison that accepts either on either side. This supplies all four arities from one user-provided key ordering.

## `AssociativeVectorComparer`

**Contract** — given an ordering on keys, answer the same question for (key, key), (pair, pair), (pair, key) and (key, pair), in every case by comparing the keys. Stateless unless the supplied ordering carries state, in which case it is copied in once at construction. Never allocates, never fails.

```text
compare(a, b) -> bool          # a is ordered before b
  key_of(a) ORDERED BEFORE key_of(b)     # key_of(pair) = its first field;
                                         # key_of(key)  = itself
```

**Invariants** — all four forms must agree, because the container's sort uses the pair/pair form and its searches use the mixed forms; a disagreement makes a binary search land in the wrong half of a correctly sorted array, silently. In particular the supplied key ordering must be a strict weak order: the container's uniqueness invariant is "neither key is ordered before the other", and an inconsistent comparison turns that into a wrong answer rather than an error.

**Notes** — deriving from the supplied ordering rather than holding one is the C++ way to pay nothing for a stateless comparison. That is incidental. What survives is the requirement above: one ordering, four call shapes, all consistent.
