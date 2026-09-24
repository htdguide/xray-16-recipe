# src/xrCore/xrsharedmem.cpp

> The blob interner: the same idea as the string interner, applied to arbitrary byte arrays — skinning weight tables, vertex arrays, animation keys that many models share.

**Needs** — [`xrsharedmem.h`](xrsharedmem.h.md) · [`xrMemory.h`](xrMemory.h.md) · [`crc32.cpp`](crc32.cpp.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrsharedmem.h`](xrsharedmem.h.md)
**Tier floor** — T1: a record is a header followed immediately by its payload in one allocation, handed out as a typed pointer into that payload.

## Purpose

Game data repeats itself at the array level, not just the string level. Many models share an identical bone-weight block; many animations share identical key arrays; many meshes share an identical index buffer. Loading each occurrence separately costs both the memory and the load time. This table gives one canonical copy per distinct byte sequence, keyed by checksum and length.

It is a separate table from the string interner because its keys are supplied by the caller — the caller already has a checksum from having read the data — and because its records are large and few, where the string table's are small and many. That difference is why this one is an ordered array and the other is a hash table.

## State

```text
RECORD SharedBlob                 # header and payload in one allocation
  ref_count : int (32-bit)
  checksum  : int (32-bit)        # supplied by the caller, not computed here
  length    : int (32-bit)        # payload length in bytes
  padding   : int (32-bit)        # forces the payload to a 16-byte offset
  payload   : bytes

RECORD BlobTable
  lock    : mutex
  entries : list<SharedBlob>      # invariant: sorted by (checksum, length)
```

**Invariants**

- **The payload starts 16 bytes into the record.** The explicit padding field exists for exactly that: the payloads are vertex and weight arrays that are read with 4-wide float instructions, and those want 16-byte alignment. This is the whole reason the header is four fields and not three.
- The entry list is sorted by checksum then length, and is binary-searched. Equal keys are adjacent, so an exact match is found by scanning forward from the lower bound while the key still matches.
- Equality is **byte equality**, not checksum equality. The checksum narrows the search; the comparison decides.
- A record's address is stable for its whole life; handles are bare pointers.

## `dock`

**Contract** — Takes a checksum, a length and a pointer to that many bytes, and returns the canonical record, creating it if absent. All three arguments must be non-empty. Takes the lock for its whole duration. Does **not** increment the reference count — the handle does. Allocates only on a miss.

```text
FUNCTION dock(checksum, length, bytes) -> SharedBlob
  REQUIRE checksum, length and bytes are all present
  LOCK table DURING
    place = lower_bound(entries, key = (checksum, length))
    # Scan forward through the equal-key run looking for a byte match.
    candidate = place
    WHILE candidate is within entries
          AND candidate.checksum == checksum
          AND candidate.length == length
      IF bytes_equal(candidate.payload, bytes, length) THEN RETURN candidate
      candidate = next(candidate)

    record = allocate(header_size + length)     # header_size is 16
    record.ref_count = 0
    record.checksum  = checksum
    record.length    = length
    copy bytes into record.payload
    insert record at `place`                    # keeps the sort order
    RETURN record
```

**Notes** — Insertion is at the lower bound, into the middle of an array. That is a linear move per insertion, and it is accepted because this table holds thousands of entries, not millions, and insertions all happen during level load. A rebuild may use any ordered or hashed structure; what must survive is that lookup is *by checksum and length first, bytes second*, because the caller has the checksum for free and the payloads are large enough that comparing them unconditionally would dominate.

## `clean`

**Contract** — Frees every record with a zero reference count and compacts them out of the list. Holds the lock. Like the string interner, this is the only path that frees, so a blob that goes unreferenced stays resident until something sweeps.

**Notes** — The implementation frees the record and then removes the now-dangling entries in a second pass. The two steps must not be reordered: removing first loses the pointer to free.

## `stat_economy`

**Contract** — Reports, in kilobytes, the memory interning has saved: for each record, `(ref_count - 1) * length`, less the table's own overhead. Reported when the process runs out of memory.

**Notes** — The per-entry overhead subtracted is a hard-coded estimate of a container node's size plus the record header, carried over from an earlier container choice. It is an approximation and is documented as needing refactoring; a rebuild should compute its own overhead or drop the subtraction. The number is a diagnostic, never a decision input.

## `dump`

**Contract** — Writes reference count, checksum and length per record to a fixed debug path. Diagnostic only; the payloads are not dumped.

## Handle semantics

**Contract** — A typed handle over a record: create from (checksum, length, typed pointer) — the length is multiplied by the element size — copy, assign, release, index an element, ask for the element count, swap, compare, and read the reference count.

**Invariants** — The element count is the byte length divided by the element size, so a handle's type must match the type the blob was created with. Nothing records or checks the element type; a handle created as one type and read as another silently reinterprets. That is the cost of a byte-level table and a rebuild should consider carrying the element type in the record.

Release decrements and drops the pointer without freeing, exactly as the string interner does. The count is not atomic.
