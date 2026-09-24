# src/Common/object_cloner.h

> Deep-copies a value, duplicating everything it owns and sharing everything it merely borrows.

**Needs** — [`object_type_traits.h`](object_type_traits.h.md) · [`xrCore/xrMemory.h`](../xrCore/xrMemory.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`object_broker.h`](object_broker.h.md)
**Tier floor** — T1: it allocates duplicate storage for owned buffers and objects, and decides ownership by hand.

## Purpose

Some entity state has to be forked — a template record instantiated per spawn, a
configuration block copied before it is mutated, an inventory snapshot taken before a
transaction. A shallow copy of such a record leaves two owners of one buffer; a blanket
deep copy duplicates shared string storage that is reference-counted precisely so it is not
duplicated. This file draws the line between the two, using the same ownership convention
as [`object_destroyer.h`](object_destroyer.h.md), and then applies it recursively.

Clone and destroy are duals and must agree on that line: anything clone duplicates, destroy
must free; anything clone shares, destroy must leave alone.

## State

Stateless.

## `clone`

**Contract** — takes a source value and a destination of the same type and overwrites the
destination with a deep copy. Allocates for every owned buffer and every pointed-to object
it encounters. Does not free what the destination previously held — the caller is
responsible for that, and cloning over a live destination leaks.

**Invariants**

- **Borrowed text is shared, owned text is duplicated.** A read-only text reference is
  copied as a reference; a writable text buffer is copied by allocating a duplicate. Same
  rule as teardown, and it must stay the same rule.
- **Shared text is shared.** The reference-counted string type is copied by taking another
  reference, never by duplicating the characters. That is the entire reason the type
  exists, and duplicating would both waste memory and break identity comparisons that the
  resource system performs by pointer.
- A pointer field is cloned into a *new* object, never aliased. The destination pointer is
  overwritten, so a non-null destination pointer leaks.

```text
FUNCTION clone(source, destination)
  # ladder, first match wins
  IF source is borrowed text
    destination <- source                    # share the reference
    RETURN
  IF source is owned text
    destination <- duplicate_storage(source) # new buffer, same bytes
    RETURN
  IF source is shared text
    destination <- source                    # take another reference
    RETURN
  IF source is a pair
    clone(source.first,  destination.first)
    clone(source.second, destination.second)
    RETURN
  IF source is a container
    empty(destination)
    FOR EACH element IN source
      fresh <- new empty element
      clone(element, fresh)
      insert_into(destination, fresh)        # appended for a sequence,
    RETURN                                   #   key-inserted for an associative container
  IF source is a pointer
    destination <- allocate_copy_of(dereference(source))
    clone(dereference(source), dereference(destination))
    RETURN
  destination <- source                      # a plain value copies as itself
```

**Notes** — the pointer arm does the work twice on purpose. It first allocates a *copy* of the
pointee rather than an empty one, so that any field the clone recursion does not reach —
a plain value, a private field, something the type initializes for itself — arrives
correct; then it runs the deep clone over the result to fix up the owned fields the copy
shared. Allocating an empty object instead would require every cloneable type to be
constructible with no arguments and would lose anything the recursion does not know about.

The insertion choice inside the container arm is made by the shape of the container type
rather than by asking it: a container parameterized on one type is treated as a sequence
and appended to; anything else is treated as keyed and inserted into. This is a weaker test
than the one the loader uses for the same decision — the loader asks whether the container
publishes a comparator — and the two can disagree for an unusual container. That is a real
inconsistency in the family and a rebuild should pick one test.

Adapter containers — queues, stacks and priority queues — are cloned by draining a copy of
the source into a scratch container and then into the destination, because they cannot be
iterated. The double transfer is a consequence of the access pattern, not a decision. For a
last-in-first-out stack it also happens to be what preserves order; see
[`object_saver.h`](object_saver.h.md), where the same double transfer *is* load-bearing.

The free-standing entry point additionally allows cloning a borrowed text reference into an
owned buffer — the one case where the ownership convention is deliberately crossed, because
the caller is explicitly taking ownership of a copy.
