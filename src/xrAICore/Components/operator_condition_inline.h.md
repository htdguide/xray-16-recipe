# src/xrAICore/Components/operator_condition_inline.h

> One property of the world bound to one value, carrying a hash derived from both so that a whole world state can be hashed by XOR-folding its properties.

**Needs** — [`operator_condition.h`](operator_condition.h.md) · [`xrCore/Math/Random32.hpp`](../../xrCore/Math/Random32.hpp.md)
**Used by** — [`operator_condition.h`](operator_condition.h.md)
**Tier floor** — T2: an immutable pair plus a deterministic scramble; the scramble's exact bit pattern is internal, so no width is frozen by a file format.

## Purpose

The atom of the world model. It exists as a type rather than a bare pair for one reason: the
hash. A world state hashes itself by XOR-folding its properties' hashes, and that only works
if each property's hash mixes the property identifier and the value *together* — otherwise two
states that disagree on one property would XOR to the same value.

## State

```text
RECORD Property
  condition : int      # the property identifier; small dense integers in practice
  value     : bool     # in the planner's instantiation; the type is open
  hash      : int (32-bit)   # derived from both at construction; never recomputed
```

**Invariants** — immutable after construction; the hash is a pure function of the other two
fields, so equal pairs always hash equally.

## Construction

**Contract** — takes a property identifier and a value, stores both, and derives the hash. The
derivation runs a small deterministic pseudo-random generator twice:

```text
FUNCTION derive_hash(condition, value) -> int (32-bit)
  h  <- scramble(seed = condition + 1)     # +1 so property 0 is not the zero seed
  h  <- h XOR scramble(seed = h + value)   # mixes the value into the property's hash
  RETURN h
```

**Notes** — a pseudo-random generator is used purely as an integer scrambler; there is nothing
random about it, and the generator must be seeded and drawn from locally so that the result
depends on nothing but the two inputs. Any good avalanching mix of the two fields substitutes
exactly; no stored data depends on these particular bits.

Feeding the value in as an *addend to the first hash* rather than as an independent seed means
the value's contribution differs per property, which is what stops two properties swapping
values from cancelling under the XOR-fold.

The `+1` on the seed is the only defensive touch: property identifier zero is a legitimate
property and a zero seed would give a degenerate sequence.

## `condition` / `value` / `hash_value`

**Contract** — the three fields, read-only.

## `operator<`

**Contract** — orders by property identifier, and by value when the identifiers are equal.

**Invariants** — the property-first order is what lets a world state binary-search by property
identifier alone, pairing the wanted identifier with any value. A rebuild that orders on the
value first breaks every merge walk in this chapter.

## `operator==`

**Contract** — both parts equal. Deliberately not hash-based: this is the fallback that makes
hash collisions harmless.
