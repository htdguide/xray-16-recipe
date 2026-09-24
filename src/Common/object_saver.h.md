# src/Common/object_saver.h

> Writes any value to a byte stream by walking its structure — the encoder half of the engine's persistence and spawn-record format.

**Needs** — [`object_interfaces.h`](object_interfaces.h.md) · [`object_type_traits.h`](object_type_traits.h.md) · [`object_loader.h`](object_loader.h.md) · [`xrCore/xrstring.h`](../xrCore/xrstring.h.md) · [`xrCommon/xr_string.h`](../xrCommon/xr_string.h.md)
**Used by** — [`object_broker.h`](object_broker.h.md) · [`object_loader.h`](object_loader.h.md)
**Tier floor** — T1: the fallback arm writes a value's memory image directly, so the encoding is the language's own layout.

## Purpose

Every entity record, every save-game payload and every spawn record in this engine is
encoded by this one recursion. Writing per-record encoders is what it exists to avoid, and
the consequence is that the on-disk and on-wire layout of the entire entity layer is
*derived from the declaration order of fields* rather than written down anywhere. That is
the single most important thing a rebuilder needs to know about the format: there is no
schema. The schema is the struct.

This file and [`object_loader.h`](object_loader.h.md) are one idea in two halves and must
be read together; they are separate only because encode and decode are separate
traversals.

## State

Stateless.

## The encoding

```text
# what each kind of value contributes to the stream

borrowed text / owned text / shared text
    -> the characters, then a zero terminator

pair
    -> first, then second                 # each filtered by the predicate

sequence, associative container, fixed-capacity vector
    -> element count as int (32-bit)
    -> each element, in iteration order

vector of truth values                    # special-cased: one bit per element
    -> element count as int (32-bit)
    -> ceil(count / 32) words of 32 bits, least significant bit first,
       nothing at all when the count is zero

queue
    -> element count as int (32-bit)
    -> elements front first

stack, priority queue
    -> element count as int (32-bit)
    -> elements top first                 # the order is reversed on the way back in

pointer
    -> whatever the pointee contributes   # no null marker exists; see invariants

value signing the persistence contract
    -> whatever its own save operation writes

anything else
    -> its memory image, verbatim
```

**Invariants**

- **The element count is always a 32-bit value**, regardless of the container's own size
  type, and is written even for an empty container. The loader reads exactly one.
- **The last arm writes a memory image.** A type that reaches it must have no hidden
  pointer in its layout — the encoder rejects a type with a dispatch table at build time
  for exactly this reason. It must also contain no owned pointers, no padding whose
  contents matter, and no field whose meaning depends on the address it was at. The
  engine's little-endian, 32-bit-`int`, 32-bit-`float` assumptions are baked into every
  byte this arm produces; see
  [§4 Platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions).
- **There is no null marker for pointers.** A pointer field is encoded as its pointee, so a
  record containing an unset pointer cannot be saved. Every pointer field that reaches this
  recursion is populated by construction, and a rebuild must either preserve that or add an
  explicit optionality marker — which changes the byte layout and therefore the format.
- **No type tags, no lengths, no versions.** Nothing in the stream says what comes next.
  Decoding requires knowing the exact type, which means the reader and writer must be the
  same build unless the type carries a version field of its own. That is why the save-game
  format is version-tagged at the outer layer and
  [refuses a mismatch rather than guessing](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence).

## `save_data`

**Contract** — takes a value, a stream and an optional *filter*, and appends the value's encoding
to the stream. Allocates only to copy adapter containers it must drain. Does not block
beyond the stream's own behaviour. The stream may be a file writer or a network packet;
the recursion does not care and only ever asks it to write raw bytes, a 32-bit value, or a
zero-terminated string.

**Invariants** — the same value written twice produces identical bytes, and the bytes are
exactly what [`object_loader.h`](object_loader.h.md) expects, element for element.

```text
FUNCTION save_data(value, stream, filter)
  # ladder, first match wins — identical in order to the loader's
  IF value is any form of text
    stream.write_terminated_string(value)
  ELSE IF value is a pair
    IF filter.admits(value, value.first, is_first = true)
      save_data(value.first, stream, filter)
    IF filter.admits(value, value.second, is_first = false)
      save_data(value.second, stream, filter)
  ELSE IF value is a vector of truth values
    save_bit_packed(value, stream)
  ELSE IF value is a container
    stream.write_u32(size(value))
    FOR EACH element IN value
      IF filter.admits(value, element)
        save_data(element, stream, filter)
  ELSE IF value is a pointer
    save_data(dereference(value), stream, filter)
  ELSE IF value signs the persistence contract
    value.save(stream)
  ELSE
    stream.write_bytes(memory image of value)
```

**Notes** — the filter is the one place where the encoding is allowed to be lossy. It is
consulted per element and per pair-half, and a rejected element is simply not written —
**but the count has already been written from the container's full size.** An encoder whose
filter rejects anything therefore produces a stream the loader will read past the end of.
Every filter in shipping use accepts everything; the ones that do not are paired with a
loader filter that compensates. This is a live trap and a rebuild should either write the
count after filtering or drop the filter.

The bit-packed encoding of a vector of truth values exists because such vectors are long
and sparse in this engine — per-level visibility and per-entity flag sets — and a byte per
element is a measurable share of a save file. It is worth keeping. Note its one asymmetry:
an empty vector writes a count and *no* words, while a vector of one writes a count and a
full word.

Last-in-first-out containers are drained top-first, which means the stream holds them in
reverse. The loader reverses again. Neither half is meaningful alone; together they are the
only way to restore a stack from a linear stream, and a rebuild that writes bottom-first
must read bottom-first.
