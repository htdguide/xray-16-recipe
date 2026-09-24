# src/xrCore/xrstring.cpp

> The string interner: every configuration key, section name, object name, bone name and texture path in the engine is a pointer into one global, checksum-bucketed, reference-counted table.

**Needs** — [`xrstring.h`](xrstring.h.md) · [`crc32.cpp`](crc32.cpp.md) · [`xrMemory.h`](xrMemory.h.md) · [`Threading/Lock.hpp`](Threading/Lock.hpp.md) · [`FS.h`](FS.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrstring.h`](xrstring.h.md)
**Tier floor** — T1: the interned record is a header followed immediately by the characters in the same allocation, and equality is pointer identity, so the record's address must be stable for its whole life. A tier whose strings may move, or whose interner it does not control, changes the equality semantics that thousands of call sites depend on.

## Purpose

The engine compares names constantly — "does this object have the bone `bip01_head`", "is this section's class identifier the one I handle" — and it loads tens of thousands of distinct names from configuration and level data. Interning makes both cheap: one copy of each distinct byte sequence exists, comparison for equality is a pointer comparison, and hashing is free because the checksum was computed at intern time and is stored in the record.

This is a separate file because the table is a process-wide singleton with its own lock, created before the filesystem and destroyed after it.

## State

```text
RECORD InternedString             # allocated as one block: header then characters
  ref_count : int (32-bit)        # invariant: the number of live handles
  length    : int (32-bit)        # invariant: == the number of characters before the terminator
  checksum  : int (32-bit)        # invariant: == crc32 over the first `length` bytes
  next      : optional<InternedString>   # chain within one bucket
  chars     : bytes               # `length` bytes plus a terminator, in the same allocation

RECORD Interner
  lock    : mutex
  buckets : list<optional<InternedString>>   # exactly 262144 entries (1024 * 256)
```

Invariants that are load-bearing and enforced nowhere except by the verify pass:

- **A record's address never changes while any handle holds it.** Every handle is a bare pointer into the table.
- **Two records in the table are never byte-equal.** That is what makes pointer equality mean string equality.
- **The bucket index is `checksum mod bucket_count`** and nothing re-buckets, so a record's bucket is fixed at insertion.
- **The header and the characters are one allocation**, so `chars` is at a fixed offset from the header and a record can be freed as a unit.

The bucket count is 262144. It is a plain power-of-two array of pointers — two megabytes of table at 64-bit pointers — sized so that a full game's name set (tens of thousands of strings) chains shallowly. Nothing measures or resizes it.

## `dock`

**Contract** — Takes a byte sequence and returns the canonical record for it, creating it if absent. Returns nothing for a nothing input (so a null handle stays null). Takes the table lock for its whole duration; the returned record's reference count is **not** incremented — that is the caller's job, and every handle does it. Allocates only on a miss.

```text
FUNCTION dock(value) -> optional<InternedString>
  IF value is none THEN RETURN none
  LOCK interner DURING
    length = length_of(value)
    sum    = crc32(value, length)

    # Search the bucket. Three-stage compare, cheapest first: the checksum
    # rejects almost everything, the length rejects the rest, and only then
    # is a byte comparison worth doing.
    candidate = buckets[sum MOD bucket_count]
    WHILE candidate is not none
      IF candidate.checksum == sum AND candidate.length == length
         AND bytes_equal(candidate.chars, value, length) THEN
        RETURN candidate
      candidate = candidate.next

    # Miss: one allocation holding header and characters, terminator included.
    record = allocate(header_size + length + 1)
    record.ref_count = 0
    record.length    = length
    record.checksum  = sum
    copy value and its terminator into record.chars
    # Push onto the front of the bucket. Insertion order within a bucket is
    # irrelevant; front insertion keeps this constant-time.
    record.next = buckets[sum MOD bucket_count]
    buckets[sum MOD bucket_count] = record
    RETURN record
```

**Notes** — A debug build has a hook that forces a *new* record for one hard-coded sentinel string, so that a suspected leak can be traced to a specific allocation. That is a diagnostic scaffold, not a behaviour: in a rebuild it is a breakpoint.

## `clean`

**Contract** — Walks every bucket and frees every record whose reference count is zero, unlinking it from its chain. Holds the lock throughout. This is the only path that frees interned strings; nothing is freed when the last handle drops, because dropping a handle only decrements.

**Invariants** — Called only when no handle is mid-creation. A record with a zero count may still be *reachable* by a caller that is about to increment it, which is why the increment happens under the same lock as the lookup.

**Notes** — Deferring the free is the interesting decision. A name that appears, disappears and reappears within one level load — which describes almost every configuration key — is interned once and reused, instead of being freed and re-allocated on each cycle. The cost is that the table only shrinks when something explicitly asks it to; the engine does so when memory runs low and between level loads.

## `verify`

**Contract** — Recomputes every record's checksum and length and fails fatally on a mismatch. This is a memory-corruption detector, not a data check: the table is written once and read forever, so a difference means something else overwrote it. Logs a start and end marker so the walk's cost is visible.

## `stat_economy`

**Contract** — Reports how many bytes interning has saved and how many distinct records exist. The saving for one record is `(ref_count - 1) * (length + 1)` — the copies that were *not* made. Reported when the process runs out of memory, which is the only place it matters.

## `dump`

**Contract** — Writes every record as reference count, length, checksum and text, either to a writer the caller supplies or to a fixed debug path. Diagnostic only.

## Handle semantics

**Contract** — A handle is a single pointer to a record, with copy, move, and assignment from raw text. Assignment from text interns first and then increments, *before* decrementing the old value — so self-assignment and assignment from a string that interns to the same record are both safe. Release decrements and then drops the pointer; it does not free.

**Invariants** — Equality, ordering and hashing are all defined on the *pointer*, not the text:

- Two handles are equal exactly when they point at the same record. Because of interning this is the same as text equality, and it is what makes name lookups in the animation and configuration layers cheap.
- Ordering is pointer ordering. It is therefore **arbitrary and not stable across runs** — it is used for keying associative containers where only consistency within one run matters, never for anything user-visible or serialized. A rebuild must not "improve" this into lexicographic ordering without checking every ordered container keyed by a name.
- The hash is the stored checksum, so hashing a name is a field read.

Comparison against a null literal is deliberately made unavailable, forcing callers to spell the emptiness test explicitly — a null handle and a handle on the empty string are different things, and conflating them was a recurring bug.

**Notes** — The reference count lives in the shared record and is **not** atomic, while the table itself is locked. That means handles may be copied concurrently only if they refer to different records — which the engine arranges by convention rather than by construction. A rebuild on a tier with real concurrency should make the count atomic; the cost is one atomic per handle copy, and handle copies are frequent.

## Case folding

**Notes** — Lowercasing a handle round-trips through a heap copy: the text is duplicated, folded, re-interned, and the copy freed. That is unavoidable given the records are immutable, and it is why the engine folds paths and section names *before* interning wherever it can. The fold itself is ASCII-only and locale-independent by requirement — a locale-aware fold changes which files match, per the platform assumptions.
