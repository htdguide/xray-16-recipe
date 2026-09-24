# src/xrEngine/Render.cpp

> The three pieces of the renderer interface that have behaviour: two resource handles that release themselves, and a scoped context switch.

**Needs** — [`Render.h`](Render.h.md) · [`Layers/xrAPI/xrAPI.h`](../Include/xrAPI/xrAPI.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Render.h`](Render.h.md)
**Tier floor** — T1: deterministic release of device resources at a defined point, and a context switch that must be exactly balanced

## Purpose

[`Render.h`](Render.h.md) is nearly all abstract. Three things in it are not, and they are
here because all three are about *ordering*: when a light is handed back, when a glow is
handed back, and how a temporary context switch is guaranteed to be undone.

## State

`Stateless.` The scoped context switch holds one saved value for its own duration.

## Light and glow release

**Contract** — when the last reference to a light or a glow goes away, the handle returns
itself to the backend. It checks that a backend still exists first, and does nothing if one
does not.

```text
FUNCTION on_last_reference_dropped(light)
  IF a renderer exists
    renderer.light_destroy(light)
```

**Notes** — the guard is the load-bearing part. At shutdown the renderer is torn down
before the last game objects are, so handles routinely outlive the thing that would free
them. Without the check, quitting the game crashes; with it, the backend has already
released everything wholesale and the handle simply evaporates. A rebuild must either keep
this ordering tolerance or invert the teardown order so the renderer outlives every handle
— the latter is cleaner and the recipe recommends it, but only if *every* handle holder can
be proven to die first, which in the original it cannot.

## `ScopedContext`

**Contract** — makes a render context current for a bounded region and restores the previous
one when the region ends, whatever path it ends by. Nests correctly, because each instance
saves the context that was current when it began rather than assuming the primary one.

```text
FUNCTION scoped_context_enter(context) -> saved
  saved = renderer.get_current_context()
  renderer.make_context_current(context)
  RETURN saved

FUNCTION scoped_context_leave(saved)
  renderer.make_context_current(saved)
```

**Invariants** — enter and leave are exactly paired; leaving restores what enter saw, not a
constant. A rebuild in a language without scope-bound cleanup must guarantee the restore on
every exit path including failure, because a leaked context switch leaves the *next* frame
recording into the helper context, which produces no output and no error.
