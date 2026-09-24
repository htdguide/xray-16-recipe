# src/Layers/xrRender/Utils/dxHashHelper.cpp

> A CRC-32 over an arbitrary run of bytes, used to give a graphics-state description a small comparable key.

**Needs** — _(none beyond the byte-width aliases)_
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it hashes a device state structure by walking its fields as raw bytes, so what it hashes is a memory layout.

## Purpose

The renderer builds far more state descriptions than there are distinct ones, and comparing two of them field by field at every lookup is wasteful. This gives each description a 32-bit key, so the state cache can bucket on the key and compare in full only on a collision. It is a separate file because it is the only piece of the graphics backends' state deduplication with no device dependency at all.

A rebuild does **not** have to use this polynomial, this construction, or even a CRC: nothing persists these values and nothing outside the process sees them. Any hash with good avalanche over a few dozen bytes will do, and the choice of a CRC here is an artifact of the era, not a requirement. The one rule that survives is the one in **Invariants**.

## State

```text
RECORD Hasher
  value : int (32-bit, wraps)    # running remainder; starts at all-ones
```

A lookup table of 256 entries is built once, on first use, and shared by every hasher thereafter.

## `Hasher` — construct, `add(bytes)`, `result() -> int`

**Contract** — a fresh hasher starts from all-ones; `add` folds a run of bytes into the running value one byte at a time; `result` returns the running value complemented. Pure, allocation-free, not thread-safe on one instance — but one instance is created per hash and never shared, so that does not matter. The shared table's first-use initialization is the one place where two threads can race; the race is benign only because every writer writes the same bytes.

**Invariants** — the hash must depend on **every byte the caller feeds it and on nothing else**. That is the real requirement, and it is easy to violate: a caller that hashes a state structure wholesale would fold in the compiler's padding, which is uninitialized, which would make two identical states hash differently and defeat the cache. The callers therefore add each field separately, and a rebuild must preserve that discipline however it hashes — either by hashing field by field, or by using a representation with no padding.

```text
FUNCTION hash_state(description) -> int
  h := new Hasher
  FOR EACH field IN description        # named, one at a time — never the whole struct
      h.add(bytes_of(field))
  RETURN h.result()
```

**Notes** — The construction is the standard reflected CRC-32: the polynomial is the one used by the common archive formats and by Ethernet, the input and output bit orders are reversed, and the value is pre-set to all-ones and complemented at the end. The engine has its own CRC-32 elsewhere, for archive checksums, and this is a second copy of the same idea living in the renderer — a duplication with no reason behind it. A rebuild should have one.
