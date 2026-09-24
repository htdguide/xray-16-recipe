# src/xrEngine/editor_helper.cpp

> One widget behaviour the debug-overlay toolkit does not provide — a collapsing header whose open/closed state survives being nested under a changing parent.

**Needs** — [`editor_helper.h`](editor_helper.h.md) · [Seam: Debug overlay UI](../../SYSTEM-REQUIREMENTS.md#seam-debug-overlay-ui)
**Used by** — [`editor_helper.h`](editor_helper.h.md)
**Tier floor** — T3: string hashing and a widget call.

## Purpose

The immediate-mode overlay identifies every widget by hashing its label together with the
enclosing scope, so two headers with the same label in different scopes are distinct. The
engine's debug panels need the opposite in one specific case: a header that appears inside
a list whose items are pushed as scopes, but whose open/closed state should be remembered
per *list*, not per *item* — otherwise expanding one entry's section and then scrolling
collapses it again.

## State

Stateless.

## `CascadingCollapsingHeader`

**Contract** — draws a collapsing header and returns whether its body should be drawn.
Identical to the toolkit's own, except that the storage key is derived from the *enclosing*
scope rather than the immediately current one. Returns false immediately when the window is
being skipped.

```text
FUNCTION cascading_collapsing_header(label, flags) -> bool
  IF the current window is skipping items THEN RETURN false

  # one level out: the parent scope, not this item's own
  IF the scope stack has more than one entry THEN
    seed = scope stack entry, second from the top
  ELSE
    seed = hash of this function's own name in the current window

  storage_key = hash(label, seed)
  set the next item's storage key to storage_key
  RETURN collapsing_header(label, flags)
```

**Notes** — the fallback seed, used when there is no enclosing scope, is an arbitrary fixed
string. Any constant works; it only has to be stable across frames and unlikely to collide.

This is a workaround for a toolkit that does not expose "state keyed by an outer scope",
and a rebuild targeting a different UI library may have no need for it at all. What is
load-bearing is the requirement: *a section's expanded state belongs to the panel, not to
the row it is currently drawn inside*.
