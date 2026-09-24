# src/xrGame/id_generator.h

> Hands out entity identifiers from a fixed range and takes them back, choosing the identifier that has been free the longest.

**Needs** — _(none)_
**Used by** — [`xrServer.h`](xrServer.h.md)
**Tier floor** — T1: a fixed-size array of fixed-size blocks, sized at compile time from the identifier range

## Purpose

Entity identifiers are a scarce, fixed-width resource: sixteen bits, packed into every
network message and every save record. When an entity dies its identifier must return to the
pool, and when a new one spawns it must get one — but **not the one that was just freed**.
A client that has not yet processed the death of entity 412 must not be handed a brand-new
entity 412, or it will apply messages for the new one to the old one.

So the allocator's real job is not "find a free number" but "find the number that has been
free the *longest*", and to do that without scanning the whole range on every allocation.

The structure is one array of fixed-size blocks. Each block is a stack of the identifiers
free within it, plus the timestamp of the most recent release into that block. Allocation
picks the block with the oldest such timestamp and pops its stack. That gives aging at block
granularity for the cost of a scan over the block count rather than the identifier count.

## State

```text
RECORD Block
  count      : int          # how many identifiers in this block are free
  time       : int          # when the MOST RECENT release into this block happened
  free       : list<int>    # exactly `block_size` slots; the first `count` are the free
                            # identifiers, stored as OFFSETS within the block, not as
                            # absolute identifiers — the block index supplies the rest

RECORD Generator
  blocks           : list<Block>   # (max - min) / block_size + 1 of them, fixed forever
  non_empty_blocks : int           # how many blocks have anything free
```

**Invariants** — the central ones:

- An identifier is stored as its offset within its block. The absolute value is
  `min + block_index * block_size + offset`, and the block index is recovered from the
  block's address. This halves the storage for a sixteen-bit range, and it is why the block
  size and the value range are compile-time constants.
- `non_empty_blocks` counts **blocks**, not identifiers. It is incremented only on a
  transition from empty to non-empty and decremented only on the reverse. It is a fast
  "is anything left" test, not a free count.
- A block's timestamp is the time of the most recent *release* into it, so ordering blocks by
  timestamp orders them by how recently they were disturbed. The block least recently
  disturbed holds the identifiers that have been free the longest.

## Block ordering

The comparison that drives the choice, stated exactly because it has two special cases:

```text
FUNCTION block_a_before_block_b(a, b) -> bool
  IF a is empty THEN RETURN false        # an empty block is never preferred
  IF b is empty THEN RETURN true         # ... and is never preferred over a non-empty one
  RETURN a.time < b.time                 # otherwise: older disturbance wins
```

**Invariants** — the empty-block cases are what let the minimum search run over the whole
array without filtering it first. A rebuild that writes the naive timestamp comparison will
hand out identifiers from empty blocks.

## Construction

**Contract** — every identifier in the range is released at a common start time, then each
block's stack is **reversed**.

```text
FUNCTION construct()
  non_empty_blocks = 0
  FOR value FROM min TO max INCLUSIVE
    free(value, start_time)
  FOR EACH block
    reverse(block.free[0 .. block.count])
```

**Invariants** — the reversal is not cosmetic. Releasing ascending values pushes them onto
each block's stack in ascending order, so popping would yield them descending. Reversing
makes a fresh generator hand out identifiers in ascending order from the low end, which is
what every save file, every log and every debugging session in the project assumes. A rebuild
that skips it produces functionally correct but unrecognizable identifier assignment.

The loop's termination test is at the *end* of the body, so the maximum value is included.
Off by one here and the top identifier is never allocatable — or, if the range's maximum is
also the invalid sentinel, the sentinel becomes allocatable.

## `tfGetID`

**Contract** — allocates an identifier. Two modes.

```text
FUNCTION get_id(requested = invalid) -> int
  IF requested is a real value THEN
    block = block_containing(requested)      # bounds-checked; out of range is fatal
    RETURN take_from(block, requested)       # fatal if it is not currently free
  ELSE
    IF non_empty_blocks = 0 THEN FAIL WITH "not enough identifiers"
    block = the block that compares least under the ordering above
    RETURN take_from(block, any)
```

**Invariants** — the specific-value mode is what a *load* uses: a save records exact
identifiers and they must be reclaimed exactly. Asking for an identifier that is already in
use is a fatal error, not a fallback, because silently substituting a different one breaks
every reference in the save.

Exhaustion is fatal too. The range is sized so that it cannot happen in a well-formed world,
and a world that exhausts it has leaked identifiers.

### Taking from a block

```text
FUNCTION take_from(block, requested) -> int
  IF block.count = 1 THEN non_empty_blocks -= 1     # this take empties the block

  IF requested is invalid THEN
    block.count -= 1
    RETURN min + block_index * block_size + block.free[block.count]   # pop the stack top
  ELSE
    offset = (requested - min) MOD block_size
    position = find offset among block.free[0 .. block.count]
    FAIL IF not found WITH "identifier already in use"
    block.count -= 1
    block.free[position] = block.free[block.count]   # swap the last entry into the hole
    RETURN requested
```

**Invariants** — the swap-with-last removal is what keeps the free set contiguous without
shifting. It **reorders the stack**, which means a specific-value allocation perturbs the
order subsequent anonymous allocations come out in. That is acceptable because the ordering
guarantee that matters is the block-level aging, not the within-block order.

The block's timestamp is deliberately **not** updated on allocation. Only releases move a
block's timestamp forward, which is correct: taking an identifier out does not make the
remaining ones any younger.

## `vfFreeID`

**Contract** — returns an identifier to the pool, stamped with the time of release.

```text
FUNCTION free(value, time)
  block = block_containing(value)               # bounds-checked
  REQUIRE block.count < block_size              # a full block means a double release
  IF block.count = 0 THEN non_empty_blocks += 1
  REQUIRE the offset is not already present     # checked in debug builds only
  block.free[block.count] = (value - min) MOD block_size
  block.count += 1
  block.time  = time
```

**Invariants** — the caller supplies the time rather than the allocator reading a clock. That
is deliberate: the aging that matters is measured on the *game* clock the network and save
protocols use, and a release replayed during a load must carry the time it originally
happened, not the time of the load.

Releasing an identifier twice is a serious bug — it puts the same identifier in the pool
twice and two live entities will eventually share it — and the duplicate check exists only in
debug builds because it is a linear scan of the block. A rebuild whose block size is small
enough should keep it always on.

**Notes** — the aging is per *block*, not per identifier. A release stamps the whole block as
freshly disturbed, so no identifier in that block is reissued while any other non-empty block
carries an older stamp; but within a block, the just-released identifier is the stack top and
so is the *first* one out when that block is finally chosen. The reuse delay is therefore
bounded below by the number of other non-empty blocks, not by the number of other free
identifiers. Block size is a memory-versus-reuse-delay trade, and its shipped value per
instantiation is a tuning decision the recipe cannot derive.

## The compile-time parameters

The generator is parameterized rather than fixed because the engine instantiates it for
different resources.

```text
min, max         : the inclusive identifier range
block_size       : identifiers per block — the aging-granularity / scan-cost trade
invalid_value    : the sentinel meaning "any identifier will do"; defaults to `max`
start_time       : the timestamp every identifier carries before its first release
```

**Invariants** — the default sentinel is the range maximum, which means that **by default the
maximum identifier is unusable as a value**: asking for it is indistinguishable from asking
for any. An instantiation that needs the full range must supply a sentinel outside it. This
is the kind of decision the source does not state and a rebuild will otherwise get wrong.

The block count is derived, not given: the range length divided by the block size, rounded up.
A range that is not a multiple of the block size leaves the last block partly unusable, and
nothing detects that — the unusable offsets are simply never released into it, so they never
come out.
