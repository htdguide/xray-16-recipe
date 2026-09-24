# src/Common/object_comparer.h

> Compares two values of the same type structurally, under a caller-chosen relation — equality, ordering, or a logical connective — applied to their leaves.

**Needs** — [`object_type_traits.h`](object_type_traits.h.md) · [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md) · [`xrCommon/xr_stack.h`](../xrCommon/xr_stack.h.md)
**Used by** — [`object_broker.h`](object_broker.h.md)
**Tier floor** — T2: it is a structural traversal with no layout or timing requirement.

## Purpose

The engine needs to ask whether two entity records are the same — after a save/load round
trip, before sending a redundant network update, when deduplicating authored data. Writing
that comparison per record is where it goes wrong, so this file derives it from the
structure: two values of the same type are compared by comparing their corresponding
leaves, and the *relation* used on the leaves is a parameter.

That parameterization is what lets one traversal produce eight named operations — equal,
not-equal, less, less-or-equal, greater, greater-or-equal, logical-and, logical-or — from
one recursion.

## State

Stateless.

## `compare`

**Contract** — takes two values of one type and a relation, and returns a single yes/no answer.
Never allocates for ordinary values; copies adapter containers in order to drain them.
Never fails: there is no "incomparable" answer.

**Invariants**

- The traversal is a **conjunction with short-circuit**: a composite compares as true only
  if *every* corresponding pair of leaves compares true, and the first false stops the walk.
- Therefore only `equal` and `not-equal` mean what their names suggest on composites. Over
  a container, `less` means "every element is less than its counterpart", which is **not**
  a lexicographic order and is not a total order. A rebuild that needs a sortable ordering
  on entity records must write one; this is not it.
- Text is compared by a three-way comparison against zero, so on text alone the ordering
  relations do behave lexicographically. The asymmetry between text and containers is a
  hazard worth stating at every call site that uses an ordering relation.

```text
FUNCTION compare(left, right, relation) -> bool
  IF left is text (borrowed, owned or shared)
    RETURN relation(three_way_compare(left, right), 0)
  IF left is a pair
    RETURN compare(left.first,  right.first,  relation)
       AND compare(left.second, right.second, relation)
  IF left is a container
    IF size(left) != size(right)
      RETURN relation(false, true)      # see note: the mismatch answer
    FOR EACH (a, b) IN zip(left, right)
      IF NOT compare(a, b, relation)
        RETURN false
    RETURN true
  IF left is a pointer
    RETURN compare(dereference(left), dereference(right), relation)   # compares pointees
  RETURN relation(left, right)          # a plain value: hand it to the relation
```

**Notes** — the size-mismatch line is the file's one genuinely clever and genuinely obscure
decision. When two containers differ in length there is no pair of leaves to feed the
relation, so it is fed a canonical *unequal* pair — false against true — and whatever the
relation says about that is the answer. Under `equal` that yields false, under `not-equal`
true, under `less` true, under `greater` false. It gives every relation a defensible answer
without special-casing any of them. It is also indistinguishable from a bug on first
reading, so a rebuild should write the intent out: *a length mismatch is resolved by asking
the relation how it feels about two things that differ.*

A pointer is compared by its pointee, not by its address. Two records with distinct but
identical sub-objects compare equal, which is what a save-round-trip check wants and what
an identity check does not. There is no null handling: comparing a record with an unset
pointer against one with a set pointer dereferences nothing-in-particular. Every call site
is expected to have established that both sides are structurally populated, which is true
for the round-trip checks this is used for and is a trap for any other use.

The eight named relations are generated from one shape. That generation is a host-language
artifact; the list of eight is not, and `logical-and` and `logical-or` in that list are
worth noticing — they mean the leaves are truth values and the composite is their
conjunction or disjunction, which is a different intent from comparison and reuses this
traversal only because the traversal is the same shape.
