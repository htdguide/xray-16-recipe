# src/xrNetServer/ip_filter.cpp

> An allow-list of address blocks, consulted before a peer is admitted; empty means everyone
> is welcome.

**Needs** — [`ip_filter.h`](ip_filter.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md)
**Used by** — reached through its declarations in [`ip_filter.h`](ip_filter.h.md); callers name that, not this file.
**Tier floor** — T2: it packs four octets into one integer and masks it. A tier without
fixed-width integer arithmetic would have to express the masking differently but nothing else
changes.

## Purpose

An operator running a private or league server wants only known networks to reach it. This
file is that list, loaded once when the server starts hosting and consulted on every admission
request. It is deliberately the crudest possible filter — no ranges, no exclusions, no
ordering — because anything richer belongs in a firewall.

## State

```text
RECORD SubnetEntry
  address : int (32-bit)   # the block's base address, in NETWORK byte order
  mask    : int (32-bit)   # the prefix as a mask: the top `bits` bits set

RECORD SubnetFilter
  entries : list<SubnetEntry>   # invariant: sorted by (address AND mask), ascending
```

**Invariants** — the list is sorted at load and never mutated afterwards, which is what makes
the binary search below legitimate. The sort key is the *masked* address, so entries are
ordered by the block they denote rather than by the literal text they were written as.

Addresses are held here in network byte order while every other address in the module is held
in host order. The conversion happens on the query path, which is the wrong end — a rebuild
should convert at load and hold one order throughout.

## `load`

**Contract** — reads a configuration file from the writable data root, parses every line of
one named section as a block in dotted-decimal-slash-prefix form, and returns how many blocks
were accepted. A missing file or a missing section yields an empty filter, which means *allow
everyone* — the deployed default. A malformed line is reported and skipped, not fatal.

```text
FUNCTION load() -> int
  FOR EACH line IN section "subnet_list" of the filter file
    parse "a.b.c.d/bits"
    IF the parse did not yield five numbers
       OR any octet exceeds 255
       OR bits is zero
    THEN report and CONTINUE
    address = (a << 24) OR (b << 16) OR (c << 8) OR d
    mask    = all ones, shifted right by (32 - bits) and back left by (32 - bits)
    append
  sort entries by (address AND mask)
  RETURN count
```

**Notes** — a prefix length of zero is rejected. A zero-length prefix would match everything,
which is already what an empty list means, so accepting it would be an obscure way of writing
nothing; rejecting it turns a typo into a complaint.

There is no upper bound on the prefix length. A value above 32 makes the shift undefined in
the host language and produces an arbitrary mask. A rebuild must clamp it.

The address is built by shifting the octets into place in the order they were written, which
puts the first-written octet in the high bits — network order, from text. That is why the
query has to convert.

## `is_ip_present`

**Contract** — takes an address in host order and answers whether any block contains it. An
empty filter answers yes for everything.

```text
FUNCTION contains(address) -> bool
  IF entries is empty THEN RETURN true          # no list means no restriction
  probe = SubnetEntry { address reversed into network order, mask = 0 }
  RETURN binary_search(entries, probe, masked_comparator)
```

**Invariants** — the comparator is the load-bearing part, and it is the only thing in this
file worth reading twice:

```text
FUNCTION masked_comparator(left, right) -> bool
  # Whichever side carries a mask supplies it for BOTH operands.
  IF left.mask is non-zero THEN
    RETURN (left.address AND left.mask) < (right.address AND left.mask)
  ELSE
    RETURN (left.address AND right.mask) < (right.address AND right.mask)
```

The probe carries a zero mask; every stored entry carries its own. So whichever side of a
comparison is a stored block imposes *its* prefix on both operands, and the probe compares
equal to that block exactly when it falls inside it. This is what lets one binary search over
blocks of differing widths answer "does any block contain this address" — a plain comparator
could not, because the ordering an address needs depends on the width of the block it is being
compared against.

**Notes** — the trick has a limit that the code does not state: it only works while no block
*contains* another. Overlapping blocks of different widths break the ordering the search
relies on, and the answer becomes whichever block the search happened to land near. Nothing
validates against it. A rebuild wanting robustness should either normalize the list to
non-overlapping blocks at load, or use a prefix trie and stop being clever.

## `unload`

**Contract** — empties the list. Called when the server stops hosting, so a restart re-reads
the file.
