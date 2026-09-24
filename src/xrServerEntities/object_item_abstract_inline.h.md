# src/xrServerEntities/object_item_abstract_inline.h

> The two field readers of a registry entry.

**Needs** — [`object_item_abstract.h`](object_item_abstract.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3.

## Purpose

Carries the construction and the two accessors declared in
[`object_item_abstract.h`](object_item_abstract.h.md). It is a separate file because this
codebase splits every inline body out of its header by habit; the split is arbitrary and a
rebuild merges it.

## `construct` / `identifier` / `script_name`

**Contract** — store the tag and the script name; return them. No allocation, no failure.
