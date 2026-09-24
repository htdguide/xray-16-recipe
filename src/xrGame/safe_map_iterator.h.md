# src/xrGame/safe_map_iterator.h

> Declares a keyed registry that is walked round-robin across frames under a time budget, and stays correct when entries are added or removed mid-walk.

**Needs** — [`safe_map_iterator_inline.h`](safe_map_iterator_inline.h.md)
**Used by** — [`alife_level_registry.h`](alife_level_registry.h.md) · [`alife_schedule_registry.h`](alife_schedule_registry.h.md) · [`safe_map_iterator_inline.h`](safe_map_iterator_inline.h.md)
**Tier floor** — T2: an ordered keyed collection with a stable cursor and a wall-clock budget

## Purpose

The engine repeatedly needs to update a collection that is too large to update in one frame,
whose members may remove themselves *during* their own update, and whose iteration must be
fair — nobody starved, nobody updated twice per cycle. The alife object registry is the
largest consumer. This declares that pattern once.

The substance is in [`safe_map_iterator_inline.h`](safe_map_iterator_inline.h.md).

## The shape it is parameterized over

Four things vary between consumers and are worth naming, because they are configuration of
the algorithm rather than of the container:

- **the key and the stored value** — the registry is keyed and holds references, not values;
- **whether the time budget applies at all** — a consumer with a small, bounded membership
  turns it off and always completes a full pass;
- **the width of the cycle counter** — it is handed to each update so a member can tell
  which pass it is being updated on, and it must not wrap over a session; the wide default
  is what guarantees that;
- **whether the first pass is exempt from the budget** — see the inline twin; this is the
  "do the whole thing once at load" switch.

Exported units:

- `add` / `remove` — register and withdraw, each with an optional relaxed mode that tolerates
  the entry already being present or absent.
- `update` — run one budgeted pass. The centre of the type.
- `set_process_time` — set the per-pass wall-clock budget.
- `begin` — restart the cursor at the first entry and exempt the next pass from the budget.
- `objects` / `empty` / `clear` — read the registry, ask if it is empty, empty it.
