# src/xrServerEntities/script_token_list_script.cpp

> Publishes the name/number vocabulary and its pair type to scripts as `token_list` and `token`.

**Needs** — [`script_token_list.h`](script_token_list.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Registers two types. `token` is the pair itself, constructible from script with both halves
readable and writable — a script builds one to pass into the engine, or receives one from a
vocabulary the engine owns. `token_list` is the vocabulary: constructible, with add, remove
by name, clear, name-to-number and number-to-name.

## Notes

**The pair's name field is exported as directly writable**, which hands script a way to
replace a text pointer the list believes it owns. Assigning through it leaks the old text
and leaves the list holding storage it did not allocate — the destructor will still try to
release it. A rebuild whose strings are owned values closes this by construction; one that
keeps raw text should export the field read-only and route writes through the list.

**The underlying array is not exported.** Only the native side reaches the terminated array
that makes this type useful, which is correct: its shape is an engine detail and a script
holding it could break the terminator invariant.
