# src/xrGame/object_manager.h

> Declares the generic "keep a set of candidates and pick the best one" base, whose whole substance is in [`object_manager_inline.h`](object_manager_inline.h.md).

**Needs** — [`object_manager_inline.h`](object_manager_inline.h.md)
**Used by** — [`enemy_manager.cpp`](enemy_manager.cpp.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`item_manager.cpp`](item_manager.cpp.md) · [`item_manager.h`](item_manager.h.md) · [`object_manager_inline.h`](object_manager_inline.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CObjectManager`, a small reusable base for every per-creature manager whose job is
"collect the things I could act on, score them, remember the best". The enemy manager, the
item manager and the corpse manager are all built on it. The substance is entirely in the
inline file because the type is generic over what it manages.

Exported units:

- `CObjectManager` — holds the candidate list and the current winner.
- `add` — offer a candidate; accepted only if it passes the usefulness test.
- `update` — rescore everything and select the winner.
- `do_evaluate` — the scoring hook a derived manager overrides. Lower is better.
- `is_useful` — the admission hook; the default admits only what the AI can see.
- `selected` / `objects` — the winner and the whole candidate list.
- `reinit` / `reset` / `Load` / `reload` — lifecycle, mostly empty at this level.
