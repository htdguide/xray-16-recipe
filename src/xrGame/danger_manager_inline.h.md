# src/xrGame/danger_manager_inline.h

> Construction, the shallow reset, and the field accessors of the danger list.

**Needs** — [`danger_manager.h`](danger_manager.h.md)
**Used by** — [`danger_manager.h`](danger_manager.h.md)
**Tier floor** — T3: field access

## Purpose

Bodies for the trivial half of [`danger_manager.h`](danger_manager.h.md), split out for
inlining. One thing here is worth stating.

## State

Adds nothing.

## Construction

**Contract** — binds the owning creature, which is required to be present. Nothing else is
initialized here; the list, the ignore list and the time line are only defined after
`reinit` runs. A rebuild should initialize them at construction — the current arrangement
means reading `time_line` on a manager that has been constructed but not reinitialized
yields whatever was in memory.

## `reset`

**Contract** — clears the danger list and the selection, and deliberately keeps the ignore
list and the time line. This is the between-lives reset: the creature forgets what it
currently fears but not what it has decided to stop fearing.

## `selected` · `objects` · `time_line`

**Contract** — read-only views of the winner and of the whole list, and a read/write pair
for the discard cutoff. The cutoff is writable from outside because the creature's memory
update owns the ageing policy, not this manager.
