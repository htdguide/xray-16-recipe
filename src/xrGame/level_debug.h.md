# src/xrGame/level_debug.h

> Declares the development build's scratchpad for on-screen diagnostics: labels floating over objects, fixed-position text, and shapes drawn in the world.

**Needs** — [`level_debug.cpp`](level_debug.cpp.md) · [`debug_text_tree.h`](debug_text_tree.h.md)
**Used by** — [`base_monster_debug.cpp`](ai/monsters/basemonster/base_monster_debug.cpp.md) · [`level_debug.cpp`](level_debug.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CLevelDebug` and the three records it stores. The whole file exists only in a
development build; a shipping build compiles nothing from it, and every call site is
similarly compiled out. See [`level_debug.cpp`](level_debug.cpp.md) for the substance.

Exported units:

- `CLevelDebug` — three registries of diagnostic items, keyed so that each contributor owns
  its own slot, plus the structured text tree used by the monster debugger.
- `SInfoItem` · `STextItem` · `SLevelItem` — the three item records: a label attached to an
  object, a label at fixed screen coordinates, and a point, line or box in the world.
- `CItemBase` — the shared container behind all three: add, remove by text or by
  identifier, clear, and iterate, with the contents always kept in identifier order.
- `CObjectInfo` · `CTextInfo` · `CLevelInfo` — the three specializations, each adding the
  typed add and the draw.
- `object_info` · `text` · `level_info` — the accessors a contributor calls to reach its own
  slot; each exists in a form that derives the slot key from the caller's runtime type.
- `draw_object_info` · `draw_text` · `draw_level_info` · `draw_debug_text` — the four draw
  passes.
- `get_text_tree` · `log_debug_info` · `debug_info_up` · `debug_info_down` — the text tree
  and its scroll position.
- `on_destroy_object` — drops everything attached to an object that is going away.

**Notes** — the slot key is the caller's *runtime class name* in one form and a
(pointer, class name) pair in the others. That is the file's one real idea and it is
explained in the implementation twin: it lets unrelated systems write diagnostics for the
same object without clearing each other's.
