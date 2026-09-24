# src/xrServerEntities/script_value_wrapper_inline.h

> The generic shadow cell's three bodies: seed from the table, push back, expose the address.

**Needs** — [`script_value_wrapper.h`](script_value_wrapper.h.md)
**Used by** — [`script_value_wrapper.h`](script_value_wrapper.h.md)
**Tier floor** — T2.

## Purpose

Carries the generic cell's definitions, split out of
[`script_value_wrapper.h`](script_value_wrapper.h.md) purely for readability. Construction
converts the named table field into the cell's type; write-back assigns the cell's value to
the table field; the value accessor answers the cell's address. The decisions are all in the
header's two specialized cases. This file does not exist in a rebuild.
