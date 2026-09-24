# src/Common/object_loader.h

> Reads any value back from a byte stream by walking its structure — the decoder half of the engine's persistence and spawn-record format.

**Needs** — [`object_interfaces.h`](object_interfaces.h.md) · [`object_type_traits.h`](object_type_traits.h.md) · [`object_saver.h`](object_saver.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md) · [`xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`object_broker.h`](object_broker.h.md) · [`object_saver.h`](object_saver.h.md)
**Tier floor** — T1: the fallback arm reads a value's memory image directly, and the pointer arm allocates.

## Purpose

The mirror of [`object_saver.h`](object_saver.h.md): one recursion that reconstructs a
value from the byte stream the encoder produced. Read the saver first — it states the
encoding, the invariants that make it decodable, and the traps. This twin covers the two
things decoding adds that encoding does not have: **allocation policy** and **merge
policy**.

## State

Stateless.

## `load_data`

**Contract** — takes a destination value, a stream and an optional *policy*, and overwrites the
destination from the stream. Allocates for owned text, for unset pointer fields and for
container growth. Does not validate: a stream that does not match the destination's type is
read as though it did, and the result is garbage rather than an error. The only structural
check anywhere is the count prefix, and it is trusted.

**Invariants**

- **A pointer field is reused if set and allocated if not.** This is the file's own warning
  and it is the sharpest edge in the family: loading into a record whose pointer fields are
  already populated *writes through them*, which is correct when reloading an existing
  object and is a leak-or-corruption when the caller expected a fresh one. The caller must
  know which it is doing.
- **Text is always allocated fresh.** An owned text field is read as a shared string and
  then duplicated into a private buffer, so the loaded record owns its text in the sense
  [`object_destroyer.h`](object_destroyer.h.md) requires. A borrowed text field *cannot* be
  loaded at all — reaching that arm is a hard failure, because there is nothing for a
  borrowed reference to point at.
- **The count prefix is authoritative.** Exactly that many elements are read, whatever the
  destination already contains.
- The decode ladder is in the same order as the encode ladder, arm for arm. Any rebuild
  that reorders one must reorder both.

```text
FUNCTION load_data(destination, stream, policy)
  IF destination is borrowed text
    FAIL WITH "a borrowed text reference has nothing to own"
  ELSE IF destination is owned text
    s <- stream.read_terminated_string()
    destination <- duplicate_storage(s)
  ELSE IF destination is shared text
    destination <- stream.read_terminated_string()      # interned, reference-counted
  ELSE IF destination is a pair
    IF policy.admits(destination, destination.first, is_first = true)
      load_data(destination.first, stream, policy)
    IF policy.admits(destination, destination.second, is_first = false)
      load_data(destination.second, stream, policy)
    policy.after_load(destination, stream)              # fix-up hook; see notes
  ELSE IF destination is a vector of truth values
    load_bit_packed(destination, stream, policy)
  ELSE IF destination is a container
    IF policy.may_clear()
      empty(destination)
    count <- stream.read_u32()
    REPEAT count TIMES
      element <- a fresh empty element
      load_data(element, stream, policy)
      IF policy.admits(destination, element)
        insert_into(destination, element)
  ELSE IF destination is a pointer
    IF destination is none
      destination <- allocate a fresh pointee           # otherwise reuse what is there
    load_data(dereference(destination), stream, policy)
  ELSE IF destination signs the persistence contract
    destination.load(stream)
  ELSE
    stream.read_bytes into memory image of destination
```

**Invariants (insertion)** — an element is appended when the container is a sequence and
key-inserted when it is ordered. The test is whether the container publishes a comparator,
which is how an ordered associative container is told from a sequence. This is a different
test from the one [`object_cloner.h`](object_cloner.h.md) uses for the same decision, and
the two can disagree; a rebuild should unify them.

## The policy

**Contract** — the optional third argument is not a filter but a small policy object with four
questions, and it is what turns one decoder into several loading modes:

```text
INTERFACE LoadPolicy
  FUNCTION may_clear() -> bool
        # false means APPEND: the loaded elements are merged into whatever the
        # destination already holds, instead of replacing it. This is how several
        # streams are accumulated into one container.
  FUNCTION admits(container, element) -> bool
        # a loaded element that is not admitted is read and discarded — the stream
        # position still advances, so filtering here is safe, unlike on the encode side
  FUNCTION admits(pair, half, is_first) -> bool
        # skips a half entirely: it is NOT read from the stream. Only safe when the
        # matching encoder skipped it too
  FUNCTION after_load(pair, stream)
        # runs once both halves of a pair are in, before the pair is inserted
```

**Invariants** — the default policy answers yes to everything and does nothing after load, and
it is what almost every call site uses.

**Notes** — the asymmetry between the two `admits` questions is load-bearing and easy to get
backwards. Rejecting a *container element* discards a value that has already been read, so
the stream stays aligned. Rejecting a *pair half* skips the read, so the stream only stays
aligned if the encoder skipped the same half. The first is a filter; the second is a schema
change.

`after_load` exists for one shape of problem: an associative container's key is read-only
once inserted, so a pair destined for a map is assembled outside the map and may need a
fix-up — resolving an identifier into a reference, say — before it is keyed. The hook is
the only place in the family where a caller can inject meaning into the traversal.

**Notes (bit-packed vectors)** — the decoder appends: it grows the destination by the stream's
count and fills only the new region, which is what makes append-mode loading work for this
type too. It reads one 32-bit word for every 32 elements, least significant bit first,
matching the encoder exactly.

## Notes

The pair arm forbids the first half from being a borrowed text reference, and says so with
a debug-only check. The real constraint is broader and is worth stating plainly: **the key
half of a pair cannot be a borrowed reference**, because loading would have to point it at
storage that does not exist yet.

Adapter containers — queues, stacks, priority queues — are read into a scratch container
and then transferred, which restores the order the encoder reversed. For a first-in-first-out
queue the transfer is order-preserving and the scratch container is redundant; for a
last-in-first-out stack it is the whole point. See the note in
[`object_saver.h`](object_saver.h.md).
