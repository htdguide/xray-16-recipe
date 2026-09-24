# src/xrGame/ef_storage_inline.h

> Choosing which of the two parameter blocks is live, by clearing the other.

**Needs** — [`ef_storage.h`](ef_storage.h.md)
**Used by** — [`ef_storage.h`](ef_storage.h.md)
**Tier floor** — T3: field access

## Purpose

Three tiny definitions, one of which is the whole switching mechanism between the client-side
and alife-side evaluation modes.

## State

Adds nothing.

## `alife_evaluation`

**Contract** — selects which parameter block subsequent evaluations read from, by **clearing
the other one**. Selecting alife evaluation clears the client block; selecting client
evaluation clears the alife block.

**Invariants** — selection is by absence, not by a flag. Every function decides which block
it is reading by testing whether the client block's member slot is filled, so an unclear
block silently wins. A rebuild that keeps the two blocks should make the choice explicit
instead; a rebuild that passes inputs as arguments deletes the problem.

**Notes** — this must be called *before* the slots are filled, and is, at the top of every
script entry point. Calling it afterwards wipes the inputs.

## `non_alife` · `alife`

**Contract** — writable references to the two blocks. Writable because filling them is how a
caller passes arguments.
