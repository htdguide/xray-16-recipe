# src/Common/object_destroyer.h

> Recursively tears down a value — running explicit teardown on anything that asks for it, releasing anything it owns, and emptying anything it contains.

**Needs** — [`object_interfaces.h`](object_interfaces.h.md) · [`object_type_traits.h`](object_type_traits.h.md) · [`xrCore/xrMemory.h`](../xrCore/xrMemory.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`object_broker.h`](object_broker.h.md)
**Tier floor** — T1: it releases memory at a chosen instant and distinguishes owned from borrowed storage by hand.

## Purpose

An entity's fields are a mixture: values that need nothing, borrowed text that must not be
freed, owned text that must, pointers to objects that own further structure, containers of
any of those, and objects that have their own explicit teardown to run first. Freeing such
a field correctly means knowing which of those it is, and the knowledge is in the type, not
in the code at the call site. This file makes one call — "destroy this" — do the right
thing for all of them.

## State

Stateless.

## `delete_data`

**Contract** — takes any value and tears it down in place. Never fails, never allocates, never
blocks. It is idempotent only where the underlying operations are: containers are emptied,
so a second call is harmless; a pointer field is *not* cleared, so a second call on the
same field is a double release and is the one hazard the caller must avoid.

**Invariants** — the ownership convention for text is the family's central rule and it is
enforced here:

- **borrowed text** (a read-only text reference) is *never* freed. It points into a shared
  string table or into a literal, and freeing it corrupts something else.
- **owned text** (a writable text buffer) *is* freed.

Nothing but the writability of the reference distinguishes the two, and every entity field
that holds text has been declared with that in mind. A rebuild must carry the distinction
explicitly, because most languages do not encode it in the type.

```text
FUNCTION delete_data(value)
  # the ladder, in this order; the first matching arm wins
  IF value is borrowed text
    RETURN                                  # not ours to free
  IF value is owned text
    release(value)
    RETURN
  IF value is a pair
    delete_data(value.first)
    delete_data(value.second)
    RETURN
  IF value is a container                   # includes fixed-capacity vectors and arrays
    FOR EACH element IN value
      delete_data(element)
    empty(value)                            # a fixed array is not emptied: it has no such
    RETURN                                  #   operation, only its elements are torn down
  IF value is a pointer
    IF value is not none
      delete_data(dereference(value))       # tear down the pointee first...
    release(value)                          # ...then release its storage
    RETURN
  IF value signs the destroyable contract
    value.destroy()
    RETURN
  RETURN                                    # a plain value owns nothing
```

**Notes** — the ladder's order is the decision. Containers are tested *before* pointers so that
a container of pointers is walked rather than mistaken for one; pointers before the
destroyable contract so that a pointer to a destroyable object gets both its teardown and
its release; and the destroyable contract last so that the explicit hook is a fallback for
types with no structure the broker can see, not a replacement for walking that structure.

The last arm is the reason this operation is safe to apply blindly to every field of a
record: a plain value simply does nothing.

The entry point takes a read-only handle and writes through it anyway. That is a
convenience so call sites do not have to spell the mutability, and it is a lie in the type
system rather than a decision — a rebuild should take a mutable handle and lose nothing.

Adapter containers — last-in-first-out stacks and priority queues — are torn down through
the same element loop even though they do not publish iteration. They are reached by a
separate arm that knows their element access; the mechanism is uninteresting, the fact that
they are covered is not.
