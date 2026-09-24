# src/Common/object_type_traits.h

> The compile-time questions the object broker asks about a type before deciding how to treat it.

**Needs** — _(none)_
**Used by** — [`object_broker.h`](object_broker.h.md) · [`object_cloner.h`](object_cloner.h.md) · [`object_comparer.h`](object_comparer.h.md) · [`object_destroyer.h`](object_destroyer.h.md) · [`object_loader.h`](object_loader.h.md) · [`object_saver.h`](object_saver.h.md) · [`object_factory_impl.h`](../xrServerEntities/object_factory_impl.h.md)
**Tier floor** — T2: these are questions about a type's shape, answered before the program runs. A language with runtime reflection answers them at startup instead and loses nothing but speed.

## Purpose

Every operation in the object broker is one recursion whose branch at each node is decided
by what kind of thing the node is. This file is the list of questions that recursion asks.
It is worth reading as a specification of the broker's *taxonomy*: a type is exactly one of
pointer, container, self-serializing object, or plain value, and these predicates are how
that is decided.

The file is also the place where one decision with wide consequences is made: the container
test is **structural, not nominal**. Nothing registers as a container; a type *is* a
container if it names the four member types below. That means the broker works on container
types nobody anticipated, and that a type which happens to name those four member types for
unrelated reasons will be silently treated as a container.

## State

Stateless — everything here is answered at build time.

## The questions

**Contract** — each is a yes/no question about a type, or a type-to-type transformation. They
have no runtime existence.

```text
# --- classification
is_pointer(T)          -> bool   # T names an indirection to another type
is_reference(T)        -> bool   # T is an alias for another value
is_const(T)            -> bool   # T is a read-only view
is_void(T)             -> bool   # T carries no value
is_same(A, B)          -> bool   # A and B are the same type, ignoring read-only-ness
is_base_and_derived(B, D) -> bool
                                 # B is a strict ancestor of D; both must be class types
                                 # and B must not be D itself

# --- shape probing (the structural test)
has_member_type(T, name) -> bool # T publishes a nested type under this exact name

is_container(T)        -> bool   # T publishes ALL FOUR of: iterator, const_iterator,
                                 #   value_type, size_type

# --- transformations
strip_pointer(T)       -> T'     # the type pointed to
strip_reference(T)     -> T'     # the type aliased
strip_const(T)         -> T'     # the same type, writable
strip_no_throw(F)      -> F'     # the same function type without its no-throw promise
```

**Invariants**

- `is_base_and_derived` is *strict*: a type is not its own ancestor. The broker relies on
  this — it asks "does this derive from the persistence interface" and the interface itself
  is abstract, so the strictness never bites in practice, but a rebuild that fuses an
  interface with a concrete base would find the answer flips.
- `is_container` is a conjunction of four independent probes, and all four are required.
  Three of the four are redundant in practice — anything with an iterator has the rest —
  and the redundancy is defensive, not meaningful.
- `strip_pointer` accepts a read-only indirection as well as a writable one and yields the
  same answer, because the broker distinguishes ownership by read-only-ness *at the text
  level only* (see [`object_cloner.h`](object_cloner.h.md)) and not for general pointers.

## Notes

Two further questions are declared and deliberately not used: whether a type publishes a
`reference` or `const_reference` member type. They were part of the container test and were
removed from it; the probes remain. Their absence from the test is what lets some of the
project's own container types qualify.

A third question — whether a type publishes a `value_compare` member type, which is how an
*ordered* container is told from a *sequence* — is defined not here but inside
[`object_loader.h`](object_loader.h.md), where it is the only place it is needed. That is
an arbitrary placement and a rebuild should keep the taxonomy in one place.

`strip_no_throw` exists for an unrelated reason: the engine stores function addresses in
tables whose element type does not carry a no-throw promise, and a function that makes the
promise has a different type from one that does not. It is a pure artifact of the host
language's type system with no counterpart in the design — a rebuild deletes it.
