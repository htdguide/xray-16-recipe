# src/xrGame/stalker_animation_pair_inline.h

> The trivial half of the animation channel: field access, the stale flag, and the callback list.

**Needs** — [`stalker_animation_pair.h`](stalker_animation_pair.h.md)
**Used by** — [`stalker_animation_pair.h`](stalker_animation_pair.h.md)
**Tier floor** — T3: field access and a small list

## Purpose

Holds the accessors that are too small to be worth a call, in a separate file because the
header must stay readable. A rebuild puts them on the type and deletes this file. Two of
them carry a decision and are written up here; the rest are plain readers and writers of
the fields listed in [`stalker_animation_pair.h`](stalker_animation_pair.h.md).

## `animation(motion)`

**Contract** — sets the wanted motion and reports whether it was already the wanted one.
The stale flag is *cleared* whenever the new motion differs from the old, and otherwise
left alone.

```text
FUNCTION set_animation(motion) -> bool
  same    := (wanted == motion)
  actual  := actual AND same          # a changed motion invalidates; an unchanged one does not revalidate
  wanted  := motion
  RETURN same
```

**Invariants** — the conjunction is the whole mechanism. A caller that re-asserts the same
motion every frame does not restart it, and a caller that asks for a different motion
guarantees the next `play` will act. Nothing else in the animation layer needs a dirty bit
because this one covers every path.

## `target_matrix(matrix)` / `target_matrix()`

**Contract** — stores a destination transform for root-moving animations, or clears it. The
stored transform is held by value and pointed at by an optional reference; clearing sets
the reference to none, and the stale value is left behind deliberately so that the
debugging overlay can still print it.

**Notes** — in a rebuild this is one `optional<Transform>` and the two functions become an
assignment and a clear.

## Callback list

**Contract** — `add_callback` appends, `remove_callback` erases, `callback` finds, and
`need_update` reports whether the list is non-empty — which is what lets the manager skip
per-frame work for channels nobody is listening to. Adding a callback that is already
present, or removing one that is not, is a programming error rather than a runtime
condition; the code asserts rather than tolerating it.
