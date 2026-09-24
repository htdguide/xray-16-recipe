# src/xrServerEntities/script_rtoken_list_script.cpp

> Publishes the interned-name list to scripts as `rtoken_list`.

**Needs** — [`script_rtoken_list.h`](script_rtoken_list.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_rtoken_list.h`](script_rtoken_list.h.md)
**Tier floor** — T3.

## Purpose

Registers the type under the script name `rtoken_list`, constructible from script, with
append, remove-by-index, clear, count and get-by-index. The export is frozen by conformance
criterion 10.

## Notes

**The exported name for the size operation is `count`, not `size`.** That rename is the only
thing this file decides, and it is load-bearing only in the sense that shipped scripts spell
it that way.

**The underlying list is not exported.** A script can build and read the vocabulary but
cannot hand the native list to anything itself — the native consumers reach it directly.
That keeps the script side from holding a reference into storage it does not own.
